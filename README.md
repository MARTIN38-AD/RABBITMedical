import discord
from discord.ext import commands
import random
from datetime import datetime
import gspread
from google.oauth2.service_account import Credentials
from asyncio import TimeoutError

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix='!', intents=intents)

@bot.event
async def on_ready():
    print(f'Logged in as {bot.user}!')

async def ask_question(ctx, question, options):
    timestamp = datetime.now().strftime('%d/%m/%Y %H:%M')
    ask = "Blacklist แก๊งค์ ครอบครัว ประชาชน? (ตอบเป็นตัวเลข)"
    options_text = "\n".join([f"{i+1}. {option}" for i, option in enumerate(options)])
    await ctx.send(f"{ask}\n\n{options_text}")

    def check(msg):
        return msg.author == ctx.author and msg.channel == ctx.channel and msg.content.isdigit()

    try:
        msg = await bot.wait_for('message', check=check, timeout=30.0)
        answer = int(msg.content)
        if 1 <= answer <= len(options):
            manage = question.split(',')
            messages = f"""
## คำสั่ง Blacklist
## ประกาศในเมือง
/md ประกาศจากโรงพยาบาล : [Blacklist] {options[answer-1]} "{manage[0]}" จนกว่าจะมาติดต่อที่โรงพยาบาล [เนื่องจาก : {manage[-1]}] ด้วยรักและห่วงใยจากทีมแพทย์ RABBIT TOWN

## ประกาศใน Discord เมืองห้อง Medical > ประกาศแบล็กลิสต์
📢   | โทษแบล็คลิส |  📢 
ประกาศ [BLACKLIST] {options[answer-1]} "{manage[0]}"

สาเหตุ : {manage[-1]} โทษปรับ 20,000 IC + Blacklist 30 นาที
สถานะ : ยังไม่ได้จ่าย

จะไม่ได้รับการรักษาจนกว่าจะมาจ่ายค่าปรับ หากภายใน 7 วันไม่จ่ายค่า BLACKLIST โทษ x2

สามารถจ่ายค่า [BLACKLIST] ได้ที่โรงพยาบาล
ประกาศ ณ วันที่ {timestamp}

TAG: @Whitelist RABBIT TOWN

## ประกาศใน Discord แพทย์ห้อง Information > รายชื่อblacklist
📢   | Blacklist |  📢 
ชื่อ {options[answer-1]} "{manage[0]}"
ติดสถานะ Blacklist ไม่ชุบ ไม่ฉีด โทษปรับ 20,000 IC + Blacklist 30 นาที

สถานะ : ยังไม่ได้จ่าย
ประกาศ ณ วันที่ {timestamp}

TAG : @RABBIT TOWN

## ----------------------------------------------------

## คำสั่งปลด Blacklist
## ประกาศในประเทศ
/md ประกาศจากโรงพยาบาล : ยกเลิกประกาศ Blacklist {options[answer-1]} "{manage[0]}" ด้วยรักและห่วงใยจากทีมแพทย์ RABBIT TOWN

## ประกาศใน Discord เมืองห้อง Medical > ประกาศแบล็กลิสต์
📢  | จ่ายค่าแบล็คลิส |  📢 
ประกาศปลด [BLACKLIST] {options[answer-1]} "{manage[0]}"
จ่ายค่า [BLACKLIST] เรียบร้อยแล้ว

สามารถทำการรักษาได้ตามปกติ 
ประกาศ ณ วันที่ {timestamp}

TAG : @Whitelist RABBIT TOWN

## ประกาศใน Discord แพทย์ห้อง WELFARE > รายรับแพทย์
📢   | เงินที่ได้รับ |  📢 
ชื่อ {options[answer-1]} "{manage[0]}"

จำนวน 20,000 IC
ประกาศ ณ วันที่ {timestamp}

TAG : @RABBIT TOWN
"""
            await ctx.send(f'{messages}')
        else:
            await ctx.send('ตัวเลือกไม่ถูกต้อง! กรุณาเลือกหมายเลขที่ถูกต้อง.')

    except TimeoutError:
        await ctx.send('หมดเวลาในการตอบ!')

async def ask_medical(ctx, question):
    search_terms = question.split(',')
    SCOPES = ['https://www.googleapis.com/auth/spreadsheets']
    creds = Credentials.from_service_account_file('credentials.json', scopes=SCOPES)
    
    try:
        client = gspread.authorize(creds)
        spreadsheet = client.open_by_url('https://docs.google.com/spreadsheets/d/1GiIOYgG6ge_PZPN-e1_FGOPQo12v4RoQHX_IisZznF8/edit?gid=1345770530')
        worksheets = spreadsheet.worksheets()
        messages = ""
        matches = []
        
        for worksheet in worksheets:
            data = worksheet.get_all_values()
            for row_index, row in enumerate(data):
                for col_index, cell in enumerate(row):
                    if any(term in cell for term in search_terms):
                        prev_col = row[col_index - 1] if col_index > 0 else None
                        next_col = row[col_index + 1] if col_index < len(row) - 1 else None
                        matches.append({
                            'sheet': worksheet.title,
                            'row_index': row_index + 1,
                            'col_index': col_index + 1,
                            'cell_value': cell,
                            'prev_col': prev_col,
                            'next_col': next_col
                        })
        
        if matches:
            for match in matches:
                messages += f"Sheet '{match['sheet']}', แถวที่ {match['row_index']}:\n{match['prev_col']} {str(match['cell_value']).strip()} {match['next_col']}\n"
        else:
            await ctx.send('ไม่เจอคำค้นหา')
        
        if messages:
            await ctx.send(f'{messages}')

    except Exception as e:
        await ctx.send(f'เกิดข้อผิดพลาดในการเข้าถึง Google Sheets: {str(e)}')

@bot.command()
async def hello(ctx):
    await ctx.send(f'Hello, {ctx.author.mention}!')

@bot.command()
async def help(ctx):
    message = """
    ## คำสั่งการใช้บอท:
    - `!hello`: ทักทาย
    - `!ask <คำถาม>`: สุ่มคำตอบ Yes, No, Maybe
    - `!bl <คำถาม>`: จัดการ Blacklist
    - `!check <คำถาม>`: ค้นหากฏที่เกี่ยวข้อง
    """
    await ctx.send(message)

@bot.command()
async def ask(ctx, *, question):
    responses = ['Yes', 'No', 'Maybe', 'Definitely!']
    await ctx.send(f'{question} {ctx.author.mention}, the answer is: {random.choice(responses)}')

@bot.command()
async def bl(ctx, *, question):
    options = ["แก๊ง", "ครอบครัว", "ประชาชน"]
    await ask_question(ctx, question, options)

@bot.command()
async def check(ctx, *, question):
    await ctx.send(f'## ค้นหากฏที่เกี่ยวข้องกับ : {question}')
    await ask_medical(ctx, question)

# Replace with your actual token in a secure way
bot.run('YOUR_BOT_TOKEN')
