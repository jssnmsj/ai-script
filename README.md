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

bot = commands.Bot(command_prefix="!", intents=intents)
bot.remove_command("help")

# File to save data
DATA_FILE = "bot_data.json"

# Trackers
active_spams = {}
spam_messages = {}
user_message_counts = {}
user_balances = {}
user_multipliers = {}
user_daily = {}  # {user_id: {"last_claim": "ISO date", "streak": int}}

# Constants
CHAT_MULTIPLIER_LEVEL_1_COST = 1_000_000_000
CHAT_MULTIPLIER_LEVEL_1_VALUE = 5
COINFLIP_MAX = 2_500_000
RISK_MAX = 5_000_000
GIVE_MAX = 999 * 10**15

DAILY_BASE = 2000
DAILY_STREAK_BONUS = 1000
DAILY_100_STREAK_REWARD = 500_000_000  # 500M

# ──────────────────────────────────────────────
# Admin Check
# ──────────────────────────────────────────────
def is_admin(member: discord.Member) -> bool:
    """Check if the user has a role named 'admin' (case insensitive)"""
    return any(role.name.lower() == "admin" for role in member.roles)


# ──────────────────────────────────────────────
# Save / Load System
# ──────────────────────────────────────────────
def save_data():
    data = {
        "balances": {str(k): v for k, v in user_balances.items()},
        "multipliers": {str(k): v for k, v in user_multipliers.items()},
        "message_counts": {str(k): v for k, v in user_message_counts.items()},
        "daily": {str(k): v for k, v in user_daily.items()}
    }
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)


def load_data():
    global user_balances, user_multipliers, user_message_counts, user_daily

    if not os.path.exists(DATA_FILE):
        return

    try:
        with open(DATA_FILE, "r") as f:
            data = json.load(f)

        user_balances = {int(k): v for k, v in data.get("balances", {}).items()}
        user_multipliers = {int(k): v for k, v in data.get("multipliers", {}).items()}
        user_message_counts = {int(k): v for k, v in data.get("message_counts", {}).items()}
        user_daily = {int(k): v for k, v in data.get("daily", {}).items()}
        print("✅ Data loaded successfully!")
    except Exception as e:
        print(f"Failed to load data: {e}")


# ──────────────────────────────────────────────
# Number helpers
# ──────────────────────────────────────────────
SUFFIXES = {
    "k": 10**3,
    "m": 10**6,
    "b": 10**9,
    "t": 10**12,
    "qd": 10**15,
}

def parse_amount(text: str) -> int:
    text = text.strip().lower().replace(",", "").replace("$", "")
    if not text:
        raise ValueError("Empty amount")

    try:
        return int(float(text))
    except ValueError:
        pass

    for suffix, mult in SUFFIXES.items():
        if text.endswith(suffix):
            num_part = text[:-len(suffix)]
            return int(float(num_part) * mult)

    raise ValueError(f"Invalid amount: {text}")


def format_amount(n: int) -> str:
    if n >= 10**15:
        return f"{n / 10**15:.2f}Qd".rstrip("0").rstrip(".")
    if n >= 10**12:
        return f"{n / 10**12:.2f}T".rstrip("0").rstrip(".")
    if n >= 10**9:
        return f"{n / 10**9:.2f}B".rstrip("0").rstrip(".")
    if n >= 10**6:
        return f"{n / 10**6:.2f}M".rstrip("0").rstrip(".")
    if n >= 10**3:
        return f"{n / 10**3:.2f}K".rstrip("0").rstrip(".")
    return str(n)


def get_balance(user_id: int) -> int:
    return user_balances.get(user_id, 0)


def add_balance(user_id: int, amount: int):
    user_balances[user_id] = get_balance(user_id) + amount
    save_data()


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
# Help Command
# ──────────────────────────────────────────────
@bot.command(name="help")
async def help_command(ctx):
    embed = discord.Embed(
        title="📖 Bot Commands",
        description="List of all available commands:",
        color=discord.Color.blue()
    )

    embed.add_field(
        name="💬 Basic",
        value=(
            "`!chat [message]` → Make the bot say something\n"
            "`!spam <amount> <message>` → Spam a message (1-150)\n"
            "`!change <new message>` → Change current spam message\n"
            "`!stop` → Stop any active spam\n"
            "`!updlog` → View update log"
        ),
        inline=False
    )

    embed.add_field(
        name="🏆 Leaderboard",
        value="`!leaderboard` / `!lb` → Top 10 message leaderboard (auto-updates every 3s)",
        inline=False
    )

    embed.add_field(
        name="💰 Economy",
        value=(
            "`!balance` / `!bal` [@user] → Check balance\n"
            "`!give @user <amount>` → Transfer money\n"
            "`!daily` → Claim daily reward (streak system)\n"
            "`!upgrade chat_multiplier 1` → Upgrade chat multiplier (costs 1B)"
        ),
        inline=False
    )

    embed.add_field(
        name="🎲 Gambling",
        value=(
            "`!cf <amount>` → 50/50 coinflip (max 2.5M)\n"
            "`!cf risk <amount>` → Risk coinflip 30% win (max 5M)"
        ),
        inline=False
    )

    embed.add_field(
        name="🛡️ Admin Only",
        value=(
            "`!addmoney @user <amount>` → Give unlimited money\n"
            "`!give` (as admin) → Can give any amount without limits"
        ),
        inline=False
    )

    embed.add_field(
        name="📌 Notes",
        value="Amounts support: `K` `M` `B` `T` `Qd`\nRole name must be exactly **admin**",
        inline=False
    )

    embed.set_footer(text="Data is auto-saved • Use !help anytime")
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Update Log
# ──────────────────────────────────────────────
@bot.command(name="updlog")
async def updlog(ctx):
    embed = discord.Embed(
        title="📜 Update Log",
        description="Recent changes and improvements:",
        color=discord.Color.purple()
    )

    embed.add_field(
        name="🆕 Latest Update",
        value=(
            "• Added `!updlog` command\n"
            "• Leaderboard now updates every **3 seconds**\n"
            "• All money amounts now show with suffixes (K, M, B, T, Qd)\n"
            "• Fixed abuse bugs on give & admin commands\n"
            "• Added full Admin system (role: `admin`)"
        ),
        inline=False
    )

    embed.add_field(
        name="💰 Economy Updates",
        value=(
            "• Added `!daily` with streak system\n"
            "• Base daily: **$2,000** + **$1,000** per streak\n"
            "• **100 streak** reward: **$500,000,000**\n"
            "• Added `!addmoney` (Admin only)\n"
            "• `!give` now properly transfers money\n"
            "• Admins can give unlimited amounts"
        ),
        inline=False
    )

    embed.add_field(
        name="💾 System Updates",
        value=(
            "• Added **auto-save** system (data survives offline)\n"
            "• Saves balances, multipliers, message counts & daily streaks\n"
            "• Fixed help command conflict\n"
            "• Improved error messages across all commands"
        ),
        inline=False
    )

    embed.add_field(
        name="🎲 Other",
        value=(
            "• Coinflip + Risk coinflip\n"
            "• Chat Multiplier upgrade\n"
            "• Live leaderboard\n"
            "• Spam system with change/stop"
        ),
        inline=False
    )

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
        await ctx.send("Please specify an amount between **1** and **150**.")
        return

    if ctx.channel.id in active_spams:
        active_spams[ctx.channel.id].cancel()

    spam_messages[ctx.channel.id] = msg
    current_task = asyncio.current_task()
    active_spams[ctx.channel.id] = current_task

    try:
        for _ in range(amount):
            current_msg = spam_messages.get(ctx.channel.id, msg)
            await ctx.send(current_msg)
            await asyncio.sleep(0.5)
    except asyncio.CancelledError:
        await ctx.send("Spam stopped.")
    finally:
        if active_spams.get(ctx.channel.id) == current_task:
            del active_spams[ctx.channel.id]
            if ctx.channel.id in spam_messages:
                del spam_messages[ctx.channel.id]


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
        prefix = medals[rank - 1] if rank <= 3 else f"**#{rank}**"
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
        await ctx.send("Unknown upgrade. Only `chat_multiplier` is available.")
        return

    if level != 1:
        await ctx.send("Only level **1** is available right now.")
        return

    current_mult = user_multipliers.get(user_id, 1)
    if current_mult >= CHAT_MULTIPLIER_LEVEL_1_VALUE:
        await ctx.send(f"You already have Chat Multiplier ×{current_mult}.")
        return

    cost = CHAT_MULTIPLIER_LEVEL_1_COST
    bal = get_balance(user_id)

    if bal < cost:
        await ctx.send(
            f"Not enough money!\n"
            f"Cost: **{format_amount(cost)}**\n"
            f"Your balance: **{format_amount(bal)}**"
        )
        return

    add_balance(user_id, -cost)
    user_multipliers[user_id] = CHAT_MULTIPLIER_LEVEL_1_VALUE
    save_data()

    await ctx.send(
        f"✅ **Chat Multiplier upgraded to ×{CHAT_MULTIPLIER_LEVEL_1_VALUE}!**\n"
        f"Spent: **{format_amount(cost)}**\n"
        f"New balance: **{format_amount(get_balance(user_id))}**"
    )


@bot.command(name="balance", aliases=["bal", "money"])
async def balance(ctx, member: discord.Member = None):
    target = member or ctx.author
    bal = get_balance(target.id)
    mult = user_multipliers.get(target.id, 1)
    daily_info = user_daily.get(target.id, {"streak": 0})

    embed = discord.Embed(
        title=f"💰 {target.display_name}'s Balance",
        color=discord.Color.green()
    )
    embed.add_field(name="Cash", value=f"**{format_amount(bal)}**", inline=True)
    embed.add_field(name="Chat Multiplier", value=f"**×{mult}**", inline=True)
    embed.add_field(name="Daily Streak", value=f"**{daily_info.get('streak', 0)}** days", inline=True)
    await ctx.send(embed=embed)


@bot.command(name="give")
async def give(ctx, member: discord.Member, amount: str):
    """Transfer money. Admins can give unlimited amounts."""
    if member.id == ctx.author.id and not is_admin(ctx.author):
        await ctx.send("You can't give money to yourself.")
        return

    try:
        value = parse_amount(amount)
    except ValueError:
        await ctx.send("Invalid amount. Use numbers like `1000`, `2.5M`, `1B`, `500k`")
        return

    if value <= 0:
        await ctx.send("Amount must be positive.")
        return

    # Admin can give any amount (no limits)
    if is_admin(ctx.author):
        add_balance(member.id, value)
        await ctx.send(
            f"🛡️ **Admin Give**\n"
            f"✅ Gave **{format_amount(value)}** to {member.mention}\n"
            f"Their new balance: **{format_amount(get_balance(member.id))}**"
        )
        return

    # Normal user limits
    if value > GIVE_MAX:
        await ctx.send(f"Maximum you can give is **{format_amount(GIVE_MAX)}**.")
        return

    giver_bal = get_balance(ctx.author.id)
    if giver_bal < value:
        await ctx.send(f"You only have **{format_amount(giver_bal)}**.")
        return

    add_balance(ctx.author.id, -value)
    add_balance(member.id, value)

    await ctx.send(
        f"✅ {ctx.author.mention} gave **{format_amount(value)}** to {member.mention}\n"
        f"Your new balance: **{format_amount(get_balance(ctx.author.id))}**"
    )


@bot.command(name="addmoney")
async def addmoney(ctx, member: discord.Member, amount: str):
    """Admin only: Give unlimited money to anyone (including yourself)"""
    if not is_admin(ctx.author):
        await ctx.send("❌ You need the **admin** role to use this command.")
        return

    try:
        value = parse_amount(amount)
    except ValueError:
        await ctx.send("Invalid amount. Use numbers like `1000`, `2.5M`, `1B`, `500k`, `10Qd`")
        return

    if value <= 0:
        await ctx.send("Amount must be positive.")
        return

    add_balance(member.id, value)

    await ctx.send(
        f"🛡️ **Admin Add Money**\n"
        f"✅ Added **{format_amount(value)}** to {member.mention}\n"
        f"New balance: **{format_amount(get_balance(member.id))}**"
    )


@bot.command(name="daily")
async def daily(ctx):
    """Claim your daily reward. Streak increases reward."""
    user_id = ctx.author.id
    now = datetime.utcnow()

    data = user_daily.get(user_id, {"last_claim": None, "streak": 0})
    last_claim_str = data.get("last_claim")
    streak = data.get("streak", 0)

    if last_claim_str:
        last_claim = datetime.fromisoformat(last_claim_str)
        time_diff = now - last_claim

        if time_diff < timedelta(hours=24):
            remaining = timedelta(hours=24) - time_diff
            hours, remainder = divmod(int(remaining.total_seconds()), 3600)
            minutes = remainder // 60
            await ctx.send(f"⏳ You can claim again in **{hours}h {minutes}m**.")
            return

        if time_diff < timedelta(hours=48):
            streak += 1
        else:
            streak = 1
    else:
        streak = 1

    reward = DAILY_BASE + (streak * DAILY_STREAK_BONUS)

    bonus_text = ""
    if streak == 100:
        reward += DAILY_100_STREAK_REWARD
        bonus_text = f"\n\n🎉 **100 STREAK BONUS!** +{format_amount(DAILY_100_STREAK_REWARD)}"

    add_balance(user_id, reward)

    user_daily[user_id] = {
        "last_claim": now.isoformat(),
        "streak": streak
    }
    save_data()

    embed = discord.Embed(
        title="📅 Daily Reward Claimed!",
        color=discord.Color.gold()
    )
    embed.add_field(name="Streak", value=f"**{streak}** days", inline=True)
    embed.add_field(name="Reward", value=f"**{format_amount(reward)}**", inline=True)
    embed.add_field(name="New Balance", value=f"**{format_amount(get_balance(user_id))}**", inline=False)

    if bonus_text:
        embed.description = bonus_text

    embed.set_footer(text="Come back tomorrow to continue your streak!")
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Gambling
# ──────────────────────────────────────────────
@bot.command(name="coinflip", aliases=["cf"])
async def coinflip(ctx, *args):
    user_id = ctx.author.id

    if len(args) == 0:
        await ctx.send("Usage:\n`!cf <amount>` (50/50, max 2.5M)\n`!cf risk <amount>` (30% win, max 5M)")
        return

    is_risk = False
    amount_str = None

    if args[0].lower() == "risk":
        is_risk = True
        if len(args) < 2:
            await ctx.send("Usage: `!cf risk <amount>`")
            return
        amount_str = args[1]
    else:
        amount_str = args[0]

    try:
        amount = parse_amount(amount_str)
    except ValueError:
        await ctx.send("Invalid amount. Examples: `500k`, `1.2M`, `2.5M`")
        return

    if amount <= 0:
        await ctx.send("Amount must be positive.")
        return

    max_bet = RISK_MAX if is_risk else COINFLIP_MAX
    if amount > max_bet:
        await ctx.send(f"Max bet for this mode is **{format_amount(max_bet)}**.")
        return

    bal = get_balance(user_id)
    if bal < amount:
        await ctx.send(f"You only have **{format_amount(bal)}**.")
        return

    add_balance(user_id, -amount)

    if is_risk:
        win = random.random() < 0.30
        mode_name = "Risk Coinflip (30% win)"
    else:
        win = random.random() < 0.50
        mode_name = "Coinflip (50/50)"

    if win:
        winnings = amount * 2
        add_balance(user_id, winnings)
        result = f"🎉 **You won!** +{format_amount(amount)} (net)"
        color = discord.Color.green()
    else:
        result = f"💀 **You lost!** -{format_amount(amount)}"
        color = discord.Color.red()

    embed = discord.Embed(title=mode_name, description=result, color=color)
    embed.add_field(name="Bet", value=format_amount(amount), inline=True)
    embed.add_field(name="New Balance", value=format_amount(get_balance(user_id)), inline=True)
    await ctx.send(embed=embed)


# Run the bot
bot.run("YOUR_BOT_TOKEN_HERE")
