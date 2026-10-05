import asyncio
import discord
from discord.ext import commands
import random

# Configure intents
intents = discord.Intents.default()
intents.message_content = True  # Required to read command text
intents.dm_messages = True      # Required to receive messages in DMs
intents.members = True          # Needed for member mentions in !give

bot = commands.Bot(command_prefix="!", intents=intents)

# Track active spam tasks and current spam text per channel/DM
active_spams = {}
spam_messages = {}

# Dictionary to track total messages sent per user: {user_id: count}
user_message_counts = {}

# Economy
user_balances = {}          # {user_id: int}
user_multipliers = {}       # {user_id: int}  (default 1)

# Constants
CHAT_MULTIPLIER_LEVEL_1_COST = 1_000_000_000      # 1B
CHAT_MULTIPLIER_LEVEL_1_VALUE = 5                 # 1 message = +5 count
COINFLIP_MAX = 2_500_000                          # 2.5M
RISK_MAX = 5_000_000                              # 5M
GIVE_MAX = 999 * 10**15                           # 999Qd


# ──────────────────────────────────────────────
# Number helpers (K M B T Qd)
# ──────────────────────────────────────────────
SUFFIXES = {
    "k": 10**3,
    "m": 10**6,
    "b": 10**9,
    "t": 10**12,
    "qd": 10**15,
}

def parse_amount(text: str) -> int:
    """Parse strings like 1.5B, 2.5M, 999Qd, 1000 into int."""
    text = text.strip().lower().replace(",", "").replace("$", "")
    if not text:
        raise ValueError("Empty amount")

    # Pure number
    try:
        return int(float(text))
    except ValueError:
        pass

    # With suffix
    for suffix, mult in SUFFIXES.items():
        if text.endswith(suffix):
            num_part = text[:-len(suffix)]
            return int(float(num_part) * mult)

    raise ValueError(f"Invalid amount: {text}")


def format_amount(n: int) -> str:
    """Pretty format large numbers with K/M/B/T/Qd."""
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


def set_balance(user_id: int, amount: int):
    user_balances[user_id] = max(0, amount)


# ──────────────────────────────────────────────
# Events
# ──────────────────────────────────────────────
@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")


@bot.event
async def on_message(message):
    # Ignore messages sent by bots
    if message.author.bot:
        return

    # Count message for leaderboard (with multiplier)
    user_id = message.author.id
    multiplier = user_multipliers.get(user_id, 1)
    user_message_counts[user_id] = user_message_counts.get(user_id, 0) + multiplier

    # Process bot commands
    await bot.process_commands(message)


# ──────────────────────────────────────────────
# Help Command
# ──────────────────────────────────────────────
@bot.command(name="help")
async def help_command(ctx):
    """Shows all available commands."""
    embed = discord.Embed(
        title="📖 Bot Commands",
        description="Here's a list of all available commands:",
        color=discord.Color.blue()
    )

    embed.add_field(
        name="💬 Basic Commands",
        value=(
            "`!chat [message]` - Make the bot say something\n"
            "`!spam <amount> <message>` - Spam a message (1-150 times)\n"
            "`!change <new message>` - Change the current spam message\n"
            "`!stop` - Stop any active spam"
        ),
        inline=False
    )

    embed.add_field(
        name="🏆 Leaderboard",
        value=(
            "`!leaderboard` / `!lb` - Show the top 10 message leaderboard\n"
            "(Auto-updates every 3 seconds)"
        ),
        inline=False
    )

    embed.add_field(
        name="💰 Economy",
        value=(
            "`!balance` / `!bal` / `!money` [@user] - Check your (or someone's) balance\n"
            "`!give @user <amount>` - Give money to another user\n"
            "`!upgrade chat_multiplier 1` - Upgrade chat multiplier (costs 1B)"
        ),
        inline=False
    )

    embed.add_field(
        name="🎲 Gambling",
        value=(
            "`!coinflip <amount>` / `!cf <amount>` - 50/50 coinflip (max 2.5M)\n"
            "`!cf risk <amount>` - Risk coinflip (30% win, max 5M)"
        ),
        inline=False
    )

    embed.add_field(
        name="📌 Notes",
        value=(
            "• Amounts support suffixes: `K`, `M`, `B`, `T`, `Qd`\n"
            "• Example: `!cf 1.5M` or `!give @user 2.5B`"
        ),
        inline=False
    )

    embed.set_footer(text="Use !help to see this menu again")
    await ctx.send(embed=embed)


# ──────────────────────────────────────────────
# Basic commands
# ──────────────────────────────────────────────
@bot.command(name="chat")
async def chat(ctx, *, msg: str = None):
    await ctx.send(msg if msg else "Hello!")


@bot.command(name="spam")
async def spam(ctx, amount: int, *, msg: str):
    """
    Usage: !spam 50 Hello world
    Sends the message 'amount' times (1 to 150).
    """
    if amount < 1 or amount > 150:
        await ctx.send("Please specify an amount between 1 and 150.")
        return

    # Cancel any active spam in this channel
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
    """
    Usage: !change new text here
    Changes the message being spammed on the fly.
    """
    if ctx.channel.id in active_spams:
        spam_messages[ctx.channel.id] = new_msg
        await ctx.send(f"Spam message changed to: {new_msg}")
    else:
        await ctx.send("No active spam running to change.")


@bot.command(name="stop")
async def stop(ctx):
    """
    Stops any active spam running in the current channel or DM.
    """
    if ctx.channel.id in active_spams:
        active_spams[ctx.channel.id].cancel()
    else:
        await ctx.send("No active spam to stop.")


# ──────────────────────────────────────────────
# Leaderboard (refreshes every 3 seconds)
# ──────────────────────────────────────────────
def build_leaderboard_embed():
    """Helper function to build the top 10 leaderboard embed."""
    embed = discord.Embed(
        title="🏆 Top 10 Message Leaderboard",
        color=discord.Color.gold()
    )

    if not user_message_counts:
        embed.description = "No messages recorded yet!"
        return embed

    # Sort users by total messages in descending order
    sorted_users = sorted(user_message_counts.items(), key=lambda item: item[1], reverse=True)[:10]

    leaderboard_text = ""
    medals = ["🥇", "🥈", "🥉"]

    for rank, (user_id, count) in enumerate(sorted_users, 1):
        prefix = medals[rank - 1] if rank <= 3 else f"**#{rank}**"
        leaderboard_text += f"{prefix} <@{user_id}> — **{count}** messages\n"

    embed.description = leaderboard_text
    embed.set_footer(text="Updates automatically every 3 seconds")
    return embed


@bot.command(name="leaderboard", aliases=["lb"])
async def leaderboard(ctx):
    """
    Displays top 10 message senders and updates every 3 seconds.
    """
    lb_message = await ctx.send(embed=build_leaderboard_embed())

    # Continuously update the leaderboard message every 3 seconds
    try:
        while True:
            await asyncio.sleep(3)
            await lb_message.edit(embed=build_leaderboard_embed())
    except asyncio.CancelledError:
        pass
    except discord.NotFound:
        # Stop updating if the message was deleted by a user
        pass


# ──────────────────────────────────────────────
# Economy / Chat Multiplier
# ──────────────────────────────────────────────
@bot.command(name="upgrade")
async def upgrade(ctx, feature: str, level: int):
    """
    Usage: !upgrade chat_multiplier 1
    Level 1 costs 1B$ and makes every message count as +5.
    """
    feature = feature.lower()
    user_id = ctx.author.id

    if feature != "chat_multiplier":
        await ctx.send("Unknown upgrade. Currently only `chat_multiplier` is available.")
        return

    if level != 1:
        await ctx.send("Only level **1** is available right now.")
        return

    current_mult = user_multipliers.get(user_id, 1)
    if current_mult >= CHAT_MULTIPLIER_LEVEL_1_VALUE:
        await ctx.send(f"You already have Chat Multiplier level 1 (×{current_mult}).")
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

    # Deduct & apply
    add_balance(user_id, -cost)
    user_multipliers[user_id] = CHAT_MULTIPLIER_LEVEL_1_VALUE

    await ctx.send(
        f"✅ **Chat Multiplier upgraded to level 1!**\n"
        f"Every message you send now counts as **+{CHAT_MULTIPLIER_LEVEL_1_VALUE}** on the leaderboard.\n"
        f"Spent: **{format_amount(cost)}**\n"
        f"New balance: **{format_amount(get_balance(user_id))}**"
    )


@bot.command(name="balance", aliases=["bal", "money"])
async def balance(ctx, member: discord.Member = None):
    """Check your (or someone else's) balance."""
    target = member or ctx.author
    bal = get_balance(target.id)
    mult = user_multipliers.get(target.id, 1)

    embed = discord.Embed(
        title=f"💰 {target.display_name}'s Balance",
        color=discord.Color.green()
    )
    embed.add_field(name="Cash", value=f"**{format_amount(bal)}**", inline=True)
    embed.add_field(name="Chat Multiplier", value=f"**×{mult}**", inline=True)
    await ctx.send(embed=embed)


@bot.command(name="give")
async def give(ctx, member: discord.Member, amount: str):
    """
    Usage: !give @User 1.5B
    Gives cash to a user (max 999Qd).
    """
    try:
        value = parse_amount(amount)
    except ValueError as e:
        await ctx.send(f"Invalid amount. Use numbers like `1000`, `2.5M`, `1B`, `999Qd`.\nError: {e}")
        return

    if value <= 0:
        await ctx.send("Amount must be positive.")
        return

    if value > GIVE_MAX:
        await ctx.send(f"Maximum you can give is **{format_amount(GIVE_MAX)}**.")
        return

    add_balance(member.id, value)

    await ctx.send(
        f"✅ Gave **{format_amount(value)}** to {member.mention}.\n"
        f"Their new balance: **{format_amount(get_balance(member.id))}**"
    )


# ──────────────────────────────────────────────
# Gambling
# ──────────────────────────────────────────────
@bot.command(name="coinflip", aliases=["cf"])
async def coinflip(ctx, *args):
    """
    Usage:
      !cf 500000          → 50/50, max 2.5M
      !cf risk 1000000    → 30% win / 70% lose, max 5M
    """
    user_id = ctx.author.id

    # ── Parse arguments ──
    if len(args) == 0:
        await ctx.send("Usage:\n`!cf <amount>` (50/50, max 2.5M)\n`!cf risk <amount>` (30% win, max 5M)")
        return

    is_risk = False
    amount_str = None

    if args[0].lower() == "risk":
        is_risk = True
        if len(args) < 2:
            await ctx.send("Usage: `!cf risk <amount>` (max 5M)")
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

    # ── Limits ──
    max_bet = RISK_MAX if is_risk else COINFLIP_MAX
    if amount > max_bet:
        await ctx.send(f"Max bet for this mode is **{format_amount(max_bet)}**.")
        return

    bal = get_balance(user_id)
    if bal < amount:
        await ctx.send(f"You only have **{format_amount(bal)}**.")
        return

    # ── Deduct bet ──
    add_balance(user_id, -amount)

    # ── Flip ──
    if is_risk:
        # 30% heads (win), 70% tails (lose)
        win = random.random() < 0.30
        mode_name = "Risk Coinflip (30% win)"
    else:
        # 50/50
        win = random.random() < 0.50
        mode_name = "Coinflip (50/50)"

    if win:
        # Win = get 2× the bet back
        winnings = amount * 2
        add_balance(user_id, winnings)
        result = f"🎉 **You won!** +{format_amount(amount)} (net)"
        color = discord.Color.green()
    else:
        result = f"💀 **You lost!** -{format_amount(amount)}"
        color = discord.Color.red()

    new_bal = get_balance(user_id)

    embed = discord.Embed(
        title=mode_name,
        description=result,
        color=color
    )
    embed.add_field(name="Bet", value=format_amount(amount), inline=True)
    embed.add_field(name="New Balance", value=format_amount(new_bal), inline=True)
    await ctx.send(embed=embed)


# Run the bot
bot.run("YOUR_BOT_TOKEN_HERE")
