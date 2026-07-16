from wcferry import Wcf, WxMsg
import pandas as pd
import sqlite3
from datetime import datetime
import re
import os

# ===================== 配置区 =====================
DB_NAME = "bill_data.db"  # 本地数据库存储账单
ADMIN_LIST = []  # 管理员wxid，留空所有人可记账
ALLOW_GROUP = True  # 是否开启群聊记账
# =================================================

# 初始化数据库
def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    # 账单表
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS bill (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        time TEXT,
        room_id TEXT,
        sender_wxid TEXT,
        sender_name TEXT,
        type TEXT,
        money REAL,
        remark TEXT
    )
    ''')
    conn.commit()
    conn.close()

# 写入账单数据库
def insert_bill(room_id, sender_wxid, sender_name, bill_type, money, remark):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO bill (time, room_id, sender_wxid, sender_name, type, money, remark)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    ''', (now, room_id, sender_wxid, sender_name, bill_type, money, remark))
    conn.commit()
    conn.close()

# 解析记账语句 兼容AJZ格式 +1000#工资 / -35#奶茶
def parse_bill_text(text: str):
    pattern = r'([+-])(\d+\.?\d*)#(.*)'
    match = re.search(pattern, text.strip())
    if not match:
        return None
    symbol, money_str, remark = match.groups()
    money = float(money_str)
    if money <= 0:
        return None
    bill_type = "收入" if symbol == "+" else "支出"
    return bill_type, money, remark.strip()

# 月度统计查询（仿AJZ /ai 本月统计）
def get_month_stat(room_id: str):
    now = datetime.now()
    current_month = now.strftime("%Y-%m")
    conn = sqlite3.connect(DB_NAME)
    df = pd.read_sql('SELECT * FROM bill WHERE room_id = ?', conn, params=(room_id,))
    conn.close()
    if df.empty:
        return "暂无任何账单记录"
    df["month"] = df["time"].str[:7]
    month_df = df[df["month"] == current_month]
    if month_df.empty:
        return f"{current_month} 暂无账单"
    
    total_in = month_df[month_df["type"] == "收入"]["money"].sum()
    total_out = month_df[month_df["type"] == "支出"]["money"].sum()
    balance = total_in - total_out

    # 分类汇总
    out_group = month_df[month_df["type"] == "支出"].groupby("remark")["money"].sum()
    text = f"===== {current_month} 账单汇总 =====\n总收入：{total_in:.2f} 元\n总支出：{total_out:.2f} 元\n本月结余：{balance:.2f} 元\n\n消费明细：\n"
    for name, val in out_group.items():
        text += f"{name}：{val:.2f}元\n"
    return text

# 导出Excel账单
def export_excel(room_id):
    conn = sqlite3.connect(DB_NAME)
    df = pd.read_sql('SELECT * FROM bill WHERE room_id = ?', conn, params=(room_id,))
    conn.close()
    save_path = f"账单_{datetime.now().strftime('%Y%m%d')}.xlsx"
    df.to_excel(save_path, index=False)
    return save_path

# 消息处理核心
def handle_msg(wcf: Wcf, msg: WxMsg):
    # 过滤自身消息、非文本消息
    if msg.is_self or msg.type != 1:
        return
    room_id = msg.roomid if msg.from_group() else msg.sender
    sender_wxid = msg.sender
    sender_info = wcf.get_user_info(sender_wxid)
    sender_name = sender_info.get("name", "未知用户")
    content = msg.content.strip()

    # 1. 记账指令 +1000#工资 / -25#午餐
    bill_res = parse_bill_text(content)
    if bill_res is not None:
        bill_type, money, remark = bill_res
        insert_bill(room_id, sender_wxid, sender_name, bill_type, money, remark)
        reply = f"✅记账成功\n类型：{bill_type}\n金额：{money} 元\n备注：{remark}"
        wcf.send_text(reply, room_id)
        return
    
    # 2. /ai 本月统计（仿AJZ AI查账）
    if content.startswith("/ai"):
        stat_text = get_month_stat(room_id)
        wcf.send_text(stat_text, room_id)
        return
    
    # 3. /导出 导出Excel账单
    if content == "/导出":
        file_path = export_excel(room_id)
        wcf.send_text(f"账单已导出至本地：{file_path}", room_id)
        return
    
    # 4. /help 帮助菜单
    if content == "/help":
        help_text = """🤖仿AJZ记账机器人指令
1. 收入：+金额#备注 例：+5000#工资
2. 支出：-金额#备注 例：-32#晚餐
3. /ai 查看本月收支统计
4. /导出 导出全部账单Excel
5. /help 查看指令"""
        wcf.send_text(help_text, room_id)
        return

if __name__ == "__main__":
    init_db()
    wcf = Wcf()
    print("记账机器人启动成功，等待微信消息...")
    wcf.enable_receiving_msg()
    while True:
        msg = wcf.get_msg()
        try:
            handle_msg(wcf, msg)
        except Exception as e:
            print("消息处理异常：", e)
