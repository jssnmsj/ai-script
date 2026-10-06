import asyncio
import discord
from discord.ext import commands
import random
import json
import os
from datetime import datetime, timedelta

# Configure intents
intents = discord.Intents.default()
intents.message_content = True
intents.dm_messages = True
intents.members = True
intents.moderation = True

bot = commands.Bot(command_prefix="!", intents=intents)
bot.remove_command("help")

DATA_FILE = "bot_data.json"

# Trackers
active_spams = {}
spam_messages = {}
user_message_counts = {}
user_balances = {}
user_multipliers = {}
user_daily = {}
user_cursed = {}          # {user_id: True/False}
user_maxcap = {}          # {user_id: 0/1/2}

# Constants
CHAT_MULTIPLIER_LEVEL_1_COST = 1_000_000_000
CHAT_MULTIPLIER_LEVEL_1_VALUE = 5
GIVE_MAX = 999 * 10**15

DAILY_BASE = 2000
DAILY_STREAK_BONUS = 1000
DAILY_100_STREAK_REWARD = 500_000_000

# Maxcap upgrades
MAXCAP_1_COST = 100_000_000          # 100M
MAXCAP_2_COST = 500_000_000          # 500M

# Default limits
DEFAULT_COINFLIP_MAX = 2_500_000     # 2.5M
DEFAULT_RISK_MAX = 5_000_000         # 5M

# Level 1 limits
MAXCAP1_COINFLIP = 5_000_000         # 5M
MAXCAP1_RISK = 10_000_000            # 10M

# Level 2 limits
MAXCAP2_COINFLIP = 10_000_000        # 10M
MAXCAP2_RISK = 25_000_000            # 25M

REWARD_MULTIPLIER = 2.5              # 2.5x reward when maxcap is owned


# ──────────────────────────────────────────────
# Admin Check
# ──────────────────────────────────────────────
def is_admin(member: discord.Member) -> bool:
    return any(role.name.lower() == "admin" for role in member.roles)


# ──────────────────────────────────────────────
# Save / Load
# ──────────────────────────────────────────────
def save_data():
    data = {
        "balances": {str(k): v for k, v in user_balances.items()},
        "multipliers": {str(k): v for k, v in user_multipliers.items()},
        "message_counts": {str(k): v for k, v in user_message_counts.items()},
        "daily": {str(k): v for k, v in user_daily.items()},
        "cursed": {str(k): v for k, v in user_cursed.items()},
        "maxcap": {str(k): v for k, v in user_maxcap.items()}
    }
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)


def load_data():
    global user_balances, user_multipliers, user_message_counts, user_daily, user_cursed, user_maxcap

    if not os.path.exists(DATA_FILE):
        return

    try:
        with open(DATA_FILE, "r") as f:
            data = json.load(f)

        user_balances = {int(k): v for k, v in data.get("balances", {}).items()}
        user_multipliers = {int(k): v for k, v in data.get("multipliers", {}).items()}
        user_message_counts = {int(k): v for k, v in data.get("message_counts", {}).items()}
        user_daily = {int(k): v for k, v in data.get("daily", {}).items()}
        user_cursed = {int(k): v for k, v in data.get("cursed", {}).items()}
        user_maxcap = {int(k): v for k, v in data.get("maxcap", {}).items()}
        print("✅ Data loaded successfully!")
    except Exception as e:
        print(f"Failed to load data: {e}")


# ──────────────────────────────────────────────
# Number helpers
# ──────────────────────────────────────────────
SUFFIXES = {
    "k": 10**3, "m": 10**6, "b": 10**9, "t": 10**12, "qd": 10**15,
}

def parse_amount(text: str) -> int:
    text = text.strip().lower().replace(",", "").replace("$", "")
    if not text:
        raise ValueError("Empty amount")

    negative = False
    if text.startswith("-"):
        negative = True
        text = text[1:]

    try:
        value = int(float(text))
        return -value if negative else value
    except ValueError:
        pass

    for suffix, mult in SUFFIXES.items():
        if text.endswith(suffix):
            num_part = text[:-len(suffix)]
            value = int(float(num_part) * mult)
            return -value if negative else value

    raise ValueError(f"Invalid amount: {text}")


def format_amount(n: int) -> str:
    sign = "-" if n < 0 else ""
    n = abs(n)
    if n >= 10**15:
        return f"{sign}{n / 10**15:.2f}Qd".rstrip("0").rstrip(".")
    if n >= 10**12:
        return f"{sign}{n / 10**12:.2f}T".rstrip("0").rstrip(".")
    if n >= 10**9:
        return f"{sign}{n / 10**9:.2f}B".rstrip("0").rstrip(".")
    if n >= 10**6:
        return f"{sign}{n / 10**6:.2f}M".rstrip("0").rstrip(".")
    if n >= 10**3:
        return f"{sign}{n / 10**3:.2f}K".rstrip("0").rstrip(".")
    return f"{sign}{n}"


def get_balance(user_id: int) -> int:
    return user_balances.get(user_id, 0)


def add_balance(user_id: int, amount: int):
    user_balances[user_id] = get_balance(user_id) + amount
    save_data()


def get_maxcap_level(user_id: int) -> int:
    return user_maxcap.get(user_id, 0)


def get_coinflip_max(user_id: int) -> int:
    level = get_maxcap_level(user_id)
    if level >= 2:
        return MAXCAP2_COINFLIP
    if level >= 1:
        return MAXCAP1_COINFLIP
    return DEFAULT_COINFLIP_MAX


def get_risk_max(user_id: int) -> int:
    level = get_maxcap_level(user_id)
    if level >= 2:
        return MAXCAP2_RISK
    if level >= 1:
        return MAXCAP1_RISK
    return DEFAULT_RISK_MAX


def get_reward_mult(user_id: int) -> float:
    return REWARD_MULTIPLIER if get_maxcap_level(user_id) >= 1 else 2.0


# ──────────────────────────────────────────────
# Events
# ──────────────────────────────────────────────
@bot.event
async def on_ready():
    load_data()
    print(f"Logged in as {bot.user}")


@bot.event
async def on_message(message):
    if message.author.bot:
        return

    user_id = message.author.id
    multiplier = user_multipliers.get(user_id, 1)
    user_message_counts[user_id] = user_message_counts.get(user_id, 0) + multiplier
    save_data()

    await bot.process_commands(message)


# ──────────────────────────────────────────────
# Help
# ──────────────────────────────────────────────
@bot.command(name="help")
async def help_command(ctx):
    embed = discord.Embed(title="📖 Bot Commands", color=discord.Color.blue())

    embed.add_field(name="💬 Basic", value=(
        "`!chat [message]`\n"
        "`!spam <amount> <message>`\n"
        "`!change <message>` / `!stop`\n"
        "`!updlog` → Update log"
    ), inline=False)

    embed.add_field(name="🏆 Leaderboard", value="`!leaderboard` / `!lb`", inline=False)

    embed.add_field(name="💰 Economy", value=(
        "`!balance` / `!bal` [@user]\n"
        "`!daily` → Daily reward + streak\n"
        "`!upgrade chat_multiplier 1`\n"
        "`!maxcap 1` / `!maxcap 2` → Increase bet limits"
    ), inline=False)

    embed.add_field(name="🎲 Gambling", value=(
        "`!cf <amount>` → 50/50\n"
        "`!cf risk <amount>` → Risk mode"
    ), inline=False)

    embed.add_field(name="🛡️ Admin Only", value=(
        "`!give @user <amount>` → Give/take money\n"
        "`!addmoney @user <amount>`\n"
        "`!curse @user` → Curse luck (15% risk win)\n"
        "`!timeout @user <time> [reason]`"
    ), inline=False)

    embed.set_footer(text="Data auto-saves • Role needed: admin")
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Update Log
# ──────────────────────────────────────────────
@bot.command(name="updlog")
async def updlog(ctx):
    embed = discord.Embed(title="📜 Update Log", color=discord.Color.purple())

    embed.add_field(name="🆕 Latest Update", value=(
        "• Added `!curse @user` (Risk becomes 15/85)\n"
        "• Added `!maxcap 1` and `!maxcap 2` upgrades\n"
        "• Added `!timeout @user <time> [reason]`\n"
        "• `!give` is now **Admin only**\n"
        "• Support negative amounts in give/addmoney\n"
        "• Better amount parsing (big & small letters)"
    ), inline=False)

    embed.add_field(name="💰 Maxcap Upgrades", value=(
        "**Level 1** (100M):\n"
        "• Normal max → 5M | Risk max → 10M\n"
        "• Win reward becomes **2.5x**\n\n"
        "**Level 2** (500M):\n"
        "• Normal max → 10M | Risk max → 25M\n"
        "• Win reward stays **2.5x**"
    ), inline=False)

    embed.add_field(name="🛡️ Admin", value=(
        "• `!give` restricted to admin role only\n"
        "• `!curse` and `!timeout` added\n"
        "• Fixed small bugs"
    ), inline=False)

    embed.set_footer(text="Thanks for using the bot!")
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Basic Commands
# ──────────────────────────────────────────────
@bot.command(name="chat")
async def chat(ctx, *, msg: str = None):
    await ctx.send(msg if msg else "Hello!")


@bot.command(name="spam")
async def spam(ctx, amount: int, *, msg: str):
    if amount < 1 or amount > 150:
        await ctx.send("Amount must be between **1** and **150**.")
        return

    if ctx.channel.id in active_spams:
        active_spams[ctx.channel.id].cancel()

    spam_messages[ctx.channel.id] = msg
    current_task = asyncio.current_task()
    active_spams[ctx.channel.id] = current_task

    try:
        for _ in range(amount):
            await ctx.send(spam_messages.get(ctx.channel.id, msg))
            await asyncio.sleep(0.5)
    except asyncio.CancelledError:
        await ctx.send("Spam stopped.")
    finally:
        if active_spams.get(ctx.channel.id) == current_task:
            del active_spams[ctx.channel.id]
            spam_messages.pop(ctx.channel.id, None)


@bot.command(name="change")
async def change(ctx, *, new_msg: str):
    if ctx.channel.id in active_spams:
        spam_messages[ctx.channel.id] = new_msg
        await ctx.send(f"Spam message changed to: **{new_msg}**")
    else:
        await ctx.send("No active spam running.")


@bot.command(name="stop")
async def stop(ctx):
    if ctx.channel.id in active_spams:
        active_spams[ctx.channel.id].cancel()
    else:
        await ctx.send("No active spam to stop.")


# ──────────────────────────────────────────────
# Leaderboard
# ──────────────────────────────────────────────
def build_leaderboard_embed():
    embed = discord.Embed(title="🏆 Top 10 Message Leaderboard", color=discord.Color.gold())

    if not user_message_counts:
        embed.description = "No messages recorded yet!"
        return embed

    sorted_users = sorted(user_message_counts.items(), key=lambda x: x[1], reverse=True)[:10]
    medals = ["🥇", "🥈", "🥉"]
    text = ""

    for rank, (user_id, count) in enumerate(sorted_users, 1):
        prefix = medals[rank-1] if rank <= 3 else f"**#{rank}**"
        text += f"{prefix} <@{user_id}> — **{format_amount(count)}** messages\n"

    embed.description = text
    embed.set_footer(text="Updates every 3 seconds")
    return embed


@bot.command(name="leaderboard", aliases=["lb"])
async def leaderboard(ctx):
    lb_message = await ctx.send(embed=build_leaderboard_embed())
    try:
        while True:
            await asyncio.sleep(3)
            await lb_message.edit(embed=build_leaderboard_embed())
    except (asyncio.CancelledError, discord.NotFound):
        pass


# ──────────────────────────────────────────────
# Economy
# ──────────────────────────────────────────────
@bot.command(name="upgrade")
async def upgrade(ctx, feature: str, level: int):
    feature = feature.lower()
    user_id = ctx.author.id

    if feature != "chat_multiplier":
        await ctx.send("Unknown upgrade. Use `!maxcap 1` or `!maxcap 2` for bet limits.")
        return

    if level != 1:
        await ctx.send("Only level **1** is available.")
        return

    if user_multipliers.get(user_id, 1) >= CHAT_MULTIPLIER_LEVEL_1_VALUE:
        await ctx.send(f"You already have Chat Multiplier ×{CHAT_MULTIPLIER_LEVEL_1_VALUE}.")
        return

    cost = CHAT_MULTIPLIER_LEVEL_1_COST
    if get_balance(user_id) < cost:
        await ctx.send(f"Not enough money!\nCost: **{format_amount(cost)}**\nBalance: **{format_amount(get_balance(user_id))}**")
        return

    add_balance(user_id, -cost)
    user_multipliers[user_id] = CHAT_MULTIPLIER_LEVEL_1_VALUE
    save_data()

    await ctx.send(f"✅ Chat Multiplier upgraded to **×{CHAT_MULTIPLIER_LEVEL_1_VALUE}**!\nSpent: **{format_amount(cost)}**")


@bot.command(name="maxcap")
async def maxcap(ctx, level: int):
    user_id = ctx.author.id
    current = get_maxcap_level(user_id)

    if level not in (1, 2):
        await ctx.send("Usage: `!maxcap 1` or `!maxcap 2`")
        return

    if current >= level:
        await ctx.send(f"You already have Maxcap level **{current}**.")
        return

    if level == 2 and current < 1:
        await ctx.send("You need **Maxcap level 1** first.")
        return

    cost = MAXCAP_1_COST if level == 1 else MAXCAP_2_COST

    if get_balance(user_id) < cost:
        await ctx.send(f"Not enough money!\nCost: **{format_amount(cost)}**\nYour balance: **{format_amount(get_balance(user_id))}**")
        return

    add_balance(user_id, -cost)
    user_maxcap[user_id] = level
    save_data()

    if level == 1:
        msg = (
            f"✅ **Maxcap Level 1** purchased!\n"
            f"• Normal max bet: **5M**\n"
            f"• Risk max bet: **10M**\n"
            f"• Win reward: **2.5x**\n"
            f"Spent: **{format_amount(cost)}**"
        )
    else:
        msg = (
            f"✅ **Maxcap Level 2** purchased!\n"
            f"• Normal max bet: **10M**\n"
            f"• Risk max bet: **25M**\n"
            f"• Win reward: **2.5x**\n"
            f"Spent: **{format_amount(cost)}**"
        )

    await ctx.send(msg)


@bot.command(name="balance", aliases=["bal", "money"])
async def balance(ctx, member: discord.Member = None):
    target = member or ctx.author
    bal = get_balance(target.id)
    mult = user_multipliers.get(target.id, 1)
    streak = user_daily.get(target.id, {}).get("streak", 0)
    maxcap_lvl = get_maxcap_level(target.id)
    cursed = user_cursed.get(target.id, False)

    embed = discord.Embed(title=f"💰 {target.display_name}'s Balance", color=discord.Color.green())
    embed.add_field(name="Cash", value=f"**{format_amount(bal)}**", inline=True)
    embed.add_field(name="Chat Mult", value=f"**×{mult}**", inline=True)
    embed.add_field(name="Daily Streak", value=f"**{streak}**", inline=True)
    embed.add_field(name="Maxcap", value=f"**Level {maxcap_lvl}**", inline=True)
    embed.add_field(name="Cursed", value="Yes 💀" if cursed else "No", inline=True)
    await ctx.send(embed=embed)


@bot.command(name="give")
async def give(ctx, member: discord.Member, amount: str):
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role to use `!give`.")
        return

    try:
        value = parse_amount(amount)
    except ValueError:
        await ctx.send("Invalid amount. Examples: `1M`, `-500k`, `2.5B`")
        return

    if value == 0:
        await ctx.send("Amount cannot be zero.")
        return

    add_balance(member.id, value)

    action = "Gave" if value > 0 else "Removed"
    await ctx.send(
        f"🛡️ **Admin Give**\n"
        f"✅ {action} **{format_amount(abs(value))}** {'to' if value > 0 else 'from'} {member.mention}\n"
        f"New balance: **{format_amount(get_balance(member.id))}**"
    )


@bot.command(name="addmoney")
async def addmoney(ctx, member: discord.Member, amount: str):
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role.")
        return

    try:
        value = parse_amount(amount)
    except ValueError:
        await ctx.send("Invalid amount.")
        return

    if value == 0:
        await ctx.send("Amount cannot be zero.")
        return

    add_balance(member.id, value)
    await ctx.send(f"🛡️ Added **{format_amount(value)}** to {member.mention}\nNew balance: **{format_amount(get_balance(member.id))}**")


@bot.command(name="daily")
async def daily(ctx):
    user_id = ctx.author.id
    now = datetime.utcnow()
    data = user_daily.get(user_id, {"last_claim": None, "streak": 0})
    last_claim_str = data.get("last_claim")
    streak = data.get("streak", 0)

    if last_claim_str:
        last_claim = datetime.fromisoformat(last_claim_str)
        diff = now - last_claim

        if diff < timedelta(hours=24):
            remaining = timedelta(hours=24) - diff
            hours, rem = divmod(int(remaining.total_seconds()), 3600)
            minutes = rem // 60
            await ctx.send(f"⏳ Come back in **{hours}h {minutes}m**.")
            return

        streak = streak + 1 if diff < timedelta(hours=48) else 1
    else:
        streak = 1

    reward = DAILY_BASE + (streak * DAILY_STREAK_BONUS)
    bonus_text = ""
    if streak == 100:
        reward += DAILY_100_STREAK_REWARD
        bonus_text = f"\n\n🎉 **100 STREAK BONUS!** +{format_amount(DAILY_100_STREAK_REWARD)}"

    add_balance(user_id, reward)
    user_daily[user_id] = {"last_claim": now.isoformat(), "streak": streak}
    save_data()

    embed = discord.Embed(title="📅 Daily Reward Claimed!", color=discord.Color.gold())
    embed.add_field(name="Streak", value=f"**{streak}** days", inline=True)
    embed.add_field(name="Reward", value=f"**{format_amount(reward)}**", inline=True)
    embed.add_field(name="New Balance", value=f"**{format_amount(get_balance(user_id))}**", inline=False)
    if bonus_text:
        embed.description = bonus_text
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Curse
# ──────────────────────────────────────────────
@bot.command(name="curse")
async def curse(ctx, member: discord.Member):
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role.")
        return

    user_cursed[member.id] = True
    save_data()
    await ctx.send(f"💀 {member.mention} has been **cursed**!\nTheir Risk coinflip is now **15% win / 85% lose**.")


@bot.command(name="uncurse")
async def uncurse(ctx, member: discord.Member):
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role.")
        return

    user_cursed[member.id] = False
    save_data()
    await ctx.send(f"✨ {member.mention} has been **uncursed**. Risk is back to 30%.")


# ──────────────────────────────────────────────
# Timeout
# ──────────────────────────────────────────────
@bot.command(name="timeout")
async def timeout(ctx, member: discord.Member, time: str, *, reason: str = "No reason provided"):
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role.")
        return

    # Parse time (supports s, m, h, d)
    time = time.lower()
    try:
        if time.endswith("s"):
            seconds = int(time[:-1])
        elif time.endswith("m"):
            seconds = int(time[:-1]) * 60
        elif time.endswith("h"):
            seconds = int(time[:-1]) * 3600
        elif time.endswith("d"):
            seconds = int(time[:-1]) * 86400
        else:
            seconds = int(time)  # assume seconds
    except ValueError:
        await ctx.send("Invalid time. Examples: `30s`, `10m`, `2h`, `1d`")
        return

    if seconds < 1 or seconds > 10246 * 3600:
        await ctx.send("Time must be between **1 second** and **10246 hours**.")
        return

    try:
        duration = timedelta(seconds=seconds)
        await member.timeout(duration, reason=reason)
        await ctx.send(f"⏰ {member.mention} has been timed out for **{time}**.\nReason: {reason}")
    except discord.Forbidden:
        await ctx.send("❌ I don't have permission to timeout this member.")
    except Exception as e:
        await ctx.send(f"Failed to timeout: {e}")


# ──────────────────────────────────────────────
# Gambling
# ──────────────────────────────────────────────
@bot.command(name="coinflip", aliases=["cf"])
async def coinflip(ctx, *args):
    user_id = ctx.author.id

    if len(args) == 0:
        await ctx.send("Usage:\n`!cf <amount>`\n`!cf risk <amount>`")
        return

    is_risk = False
    amount_str = args[0]

    if args[0].lower() == "risk":
        is_risk = True
        if len(args) < 2:
            await ctx.send("Usage: `!cf risk <amount>`")
            return
        amount_str = args[1]

    try:
        amount = parse_amount(amount_str)
    except ValueError:
        await ctx.send("Invalid amount.")
        return

    if amount <= 0:
        await ctx.send("Amount must be positive.")
        return

    max_bet = get_risk_max(user_id) if is_risk else get_coinflip_max(user_id)
    if amount > max_bet:
        await ctx.send(f"Max bet is **{format_amount(max_bet)}**.\nUse `!maxcap 1` or `!maxcap 2` to increase it.")
        return

    if get_balance(user_id) < amount:
        await ctx.send(f"You only have **{format_amount(get_balance(user_id))}**.")
        return

    add_balance(user_id, -amount)

    # Win chance
    if is_risk:
        win_chance = 0.15 if user_cursed.get(user_id, False) else 0.30
        mode_name = f"Risk Coinflip ({int(win_chance*100)}% win)"
    else:
        win_chance = 0.50
        mode_name = "Coinflip (50/50)"

    win = random.random() < win_chance
    reward_mult = get_reward_mult(user_id)

    if win:
        winnings = int(amount * reward_mult)
        add_balance(user_id, winnings)
        result = f"🎉 **You won!** +{format_amount(winnings - amount)} (net)"
        color = discord.Color.green()
    else:
        result = f"💀 **You lost!** -{format_amount(amount)}"
        color = discord.Color.red()

    embed = discord.Embed(title=mode_name, description=result, color=color)
    embed.add_field(name="Bet", value=format_amount(amount), inline=True)
    embed.add_field(name="New Balance", value=format_amount(get_balance(user_id)), inline=True)
    if get_maxcap_level(user_id) >= 1:
        embed.set_footer(text=f"Reward multiplier: {reward_mult}x")
    await ctx.send(embed=embed)


# Run the bot
bot.run("YOUR_BOT_TOKEN_HERE")
