``` import discord
from discord import app_commands
from discord.ext import commands
import datetime
import asyncio

intents = discord.Intents.all()
bot = commands.Bot(command_prefix="/", intents=intents)

def create_embed(title, description, success=True):
    color = discord.Color.green() if success else discord.Color.red()
    emb = discord.Embed(title=title, description=description, color=color, timestamp=datetime.datetime.utcnow())
    emb.set_footer(text="System Terminal Log", icon_url=bot.user.avatar.url if bot.user.avatar else None)
    return emb

@bot.event
async def on_ready():
    print(f'----BOT IS ONLINE----')
    print(f'{bot.user}')
    try:
        synced = await bot.tree.sync()
        print(f'Engine Status: Synced {len(synced)} dynamic UI commands.')
    except Exception as e:
        print(f'Engine Sync Error: {e}')
    print(f'---------------------')

class CustomTimeoutModal(discord.ui.Modal, title="Set Timeout Duration"):
    duration = discord.ui.TextInput(
        label="Enter number (e.g. 10, 30, 12)",
        placeholder="Type duration here...",
        min_length=1,
        max_length=5
    )

    def __init__(self, target_user: discord.Member):
        super().__init__()
        self.target_user = target_user

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        try:
            val = int(self.duration.value)
        except ValueError:
            return await interaction.followup.send(embed=create_embed("Error", "You must enter a valid whole number.", False))

        view = discord.ui.View()
        view.add_item(TimeoutUnitSelect(target_user=self.target_user, value=val))
        await interaction.followup.send(f"Selected duration: **{val}**. Now select the time unit below:", view=view, ephemeral=True)


class CreateChannelModal(discord.ui.Modal, title="Create Channel"):
    chan_name = discord.ui.TextInput(
        label="Channel Name",
        placeholder="Type channel name (e.g. general, announcements)...",
        min_length=1,
        max_length=100
    )

    def __init__(self, channel_type: str):
        super().__init__()
        self.channel_type = channel_type

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        name = self.chan_name.value.strip().replace(" ", "-").lower()
        try:
            if self.channel_type == "text":
                new_chan = await interaction.guild.create_text_channel(name=name)
            else:
                new_chan = await interaction.guild.create_voice_channel(name=name)
            await interaction.followup.send(embed=create_embed("Success", f"Created {self.channel_type} channel: {new_chan.mention}"))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Failed to create channel: `{e}`", False))


class CreateCategoryModal(discord.ui.Modal, title="Create Category"):
    cat_name = discord.ui.TextInput(
        label="Category Name",
        placeholder="Type category name here...",
        min_length=1,
        max_length=100
    )

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        name = self.cat_name.value.strip()
        try:
            new_cat = await interaction.guild.create_category(name=name)
            await interaction.followup.send(embed=create_embed("Success", f"Created category: **{new_cat.name}**"))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Failed to create category: `{e}`", False))


class CustomPurgeModal(discord.ui.Modal, title="Purge Messages"):
    amount_input = discord.ui.TextInput(
        label="Amount to purge (Max 100)",
        placeholder="Type a number between 1 and 100...",
        min_length=1,
        max_length=3
    )

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        try:
            amount = int(self.amount_input.value)
            if amount < 1 or amount > 100:
                raise ValueError()
        except ValueError:
            return await interaction.followup.send(embed=create_embed("Error", "Please enter a number between 1 and 100.", False))

        try:
            deleted = await interaction.channel.purge(limit=amount)
            confirmation = await interaction.followup.send(embed=create_embed("Purge Completed", f"Successfully deleted **{len(deleted)}** messages."))
            await asyncio.sleep(3)
            await confirmation.delete()
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Purge failed: `{e}`", False))


class CustomSlowmodeModal(discord.ui.Modal, title="Set Slowmode"):
    secs_input = discord.ui.TextInput(
        label="Cooldown in seconds (0 to disable)",
        placeholder="Enter number of seconds (e.g. 5, 10, 120)...",
        min_length=1,
        max_length=5
    )

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        try:
            secs = int(self.secs_input.value)
            if secs < 0:
                raise ValueError()
        except ValueError:
            return await interaction.followup.send(embed=create_embed("Error", "Please enter a valid positive number.", False))

        try:
            await interaction.channel.edit(slowmode_delay=secs)
            await interaction.followup.send(embed=create_embed("Success", f"Slowmode delay set to **{secs}** seconds."))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Action failed: `{e}`", False))

class TimeoutUnitSelect(discord.ui.Select):
    def __init__(self, target_user: discord.Member, value: int):
        self.target_user = target_user
        self.value = value
        options = [
            discord.SelectOption(label="Minutes (m)", value="m", description=f"Timeout for {value} minutes"),
            discord.SelectOption(label="Hours (h)", value="h", description=f"Timeout for {value} hours"),
            discord.SelectOption(label="Days (d)", value="d", description=f"Timeout for {value} days")
        ]
        super().__init__(placeholder="Select time unit...", options=options)

    async def callback(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        unit = self.values[0]
        
        if unit == "m":
            time_delta = datetime.timedelta(minutes=self.value)
            text = f"{self.value} minutes"
        elif unit == "h":
            time_delta = datetime.timedelta(hours=self.value)
            text = f"{self.value} hours"
        else:
            time_delta = datetime.timedelta(days=self.value)
            text = f"{self.value} days"

        try:
            await self.target_user.timeout(time_delta, reason="Master Panel Timeout")
            await interaction.followup.send(embed=create_embed("Success", f"User **{self.target_user.name}** has been timed out for **{text}**."))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Timeout failed: `{e}`", False))


class TimeoutUserFirstSelect(discord.ui.UserSelect):
    def __init__(self):
        super().__init__(placeholder="Select user for Timeout...", min_values=1, max_values=1)

    async def callback(self, interaction: discord.Interaction):
        target_user = self.values[0]
      
        await interaction.response.send_modal(CustomTimeoutModal(target_user=target_user))


class CustomUserSelect(discord.ui.UserSelect):
    def __init__(self, action: str, placeholder: str):
        super().__init__(placeholder=placeholder, min_values=1, max_values=1)
        self.action = action

    async def callback(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        target_user = self.values[0]

        try:
            if self.action == "kick":
                await target_user.kick(reason="Master Panel")
                await interaction.followup.send(embed=create_embed("Success", f"User **{target_user.name}** has been kicked."))
            
            elif self.action == "ban":
                await target_user.ban(reason="Master Panel")
                await interaction.followup.send(embed=create_embed("Success", f"User **{target_user.name}** has been banned."))
            
            elif self.action in ["role_add", "role_remove"]:
                role_view = discord.ui.View()
                role_view.add_item(CustomRoleSelect(action=self.action, target_user=target_user))
                await interaction.followup.send(f"Select a role for **{target_user.name}**:", view=role_view, ephemeral=True)
                
            elif self.action == "strip_roles":
                roles_to_remove = [r for r in target_user.roles if r != interaction.guild.default_role and r < interaction.guild.me.top_role]
                await target_user.remove_roles(*roles_to_remove)
                await interaction.followup.send(embed=create_embed("Success", f"Removed all roles from **{target_user.name}**."))
                
            elif self.action == "user_info":
                emb = discord.Embed(title=f"Profile: {target_user.name}", color=discord.Color.blue())
                emb.set_thumbnail(url=target_user.avatar.url if target_user.avatar else None)
                emb.add_field(name="ID:", value=f"`{target_user.id}`", inline=True)
                emb.add_field(name="Joined Server:", value=target_user.joined_at.strftime("%d.%m.%Y"), inline=True)
                emb.add_field(name="Highest Role:", value=target_user.top_role.mention, inline=True)
                await interaction.followup.send(embed=emb, ephemeral=True)

        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Operation failed:\n`{e}`", False))


class CustomRoleSelect(discord.ui.RoleSelect):
    def __init__(self, action: str, target_user: discord.Member = None, delete: bool = False):
        placeholder = "Select role to delete..." if delete else "Select role..."
        super().__init__(placeholder=placeholder, min_values=1, max_values=1)
        self.action = action
        self.target_user = target_user
        self.delete = delete

    async def callback(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        role = self.values[0]

        try:
            if self.delete:
                role_name = role.name
                await role.delete()
                return await interaction.followup.send(embed=create_embed("Success", f"Role **{role_name}** has been deleted."))

            if self.action == "role_add":
                await self.target_user.add_roles(role)
                await interaction.followup.send(embed=create_embed("Success", f"Role **{role.name}** added to **{self.target_user.name}**."))
            elif self.action == "role_remove":
                await self.target_user.remove_roles(role)
                await interaction.followup.send(embed=create_embed("Success", f"Role **{role.name}** removed from **{self.target_user.name}**."))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Role action failed:\n`{e}`", False))


class CustomChannelSelect(discord.ui.ChannelSelect):
    def __init__(self, action: str):
        super().__init__(placeholder="Select channel...", min_values=1, max_values=1)
        self.action = action

    async def callback(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        chan = self.values[0]
        try:
            if self.action == "delete":
                name = chan.name
                await chan.delete()
                await interaction.followup.send(embed=create_embed("Success", f"Channel **{name}** has been deleted."))
                
            elif self.action == "lock":
                await chan.set_permissions(interaction.guild.default_role, send_messages=False)
                await interaction.followup.send(embed=create_embed("Success", f"Channel {chan.mention} has been locked."))
                
            elif self.action == "unlock":
                await chan.set_permissions(interaction.guild.default_role, send_messages=None)
                await interaction.followup.send(embed=create_embed("Success", f"Channel {chan.mention} has been unlocked."))
        except Exception as e:
            await interaction.followup.send(embed=create_embed("Error", f"Channel action failed:\n`{e}`", False))


class DynamicActionView(discord.ui.View):
    def __init__(self, action: str):
        super().__init__(timeout=120)
        
        if action == "ban":
            self.add_item(CustomUserSelect(action="ban", placeholder="Select member to BAN..."))
        elif action == "kick":
            self.add_item(CustomUserSelect(action="kick", placeholder="Select member to KICK..."))
        elif action == "timeout":
            self.add_item(TimeoutUserFirstSelect())
        elif action in ["role_add", "role_remove"]:
            self.add_item(CustomUserSelect(action=action, placeholder="Select member for roles..."))
        elif action == "strip_roles":
            self.add_item(CustomUserSelect(action="strip_roles", placeholder="Select member to strip ALL roles..."))
        elif action == "user_info":
            self.add_item(CustomUserSelect(action="user_info", placeholder="Select member for information..."))
        elif action == "delete_channel":
            self.add_item(CustomChannelSelect(action="delete"))
        elif action == "lock_channel":
            self.add_item(CustomChannelSelect(action="lock"))
        elif action == "unlock_channel":
            self.add_item(CustomChannelSelect(action="unlock"))
        elif action == "role_delete":
            self.add_item(CustomRoleSelect(action="delete", delete=True))


class HelpDropdown(discord.ui.Select):
    def __init__(self):
        options = [
            discord.SelectOption(label="Moderation", description="View moderation options"),
            discord.SelectOption(label="Roles", description="View role control options"),
            discord.SelectOption(label="Channels and Chat", description="View chat and channel control options")
        ]
        super().__init__(placeholder="Select help category...", options=options)

    async def callback(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        choice = self.values[0]
        
        if "Moderation" in choice:
            desc = (
                "**Ban member** - Permanent ban.\n"
                "**Kick member** - Instant kick.\n"
                "**Timeout member** - Double menu selection for timeouts (m, h, d).\n"
                "**View info** - Profile card containing IDs and roles."
            )
            emb = discord.Embed(title="Moderation Commands", description=desc, color=discord.Color.red())
        elif "Roles" in choice:
            desc = (
                "**Add role** - Assign role to a selected member.\n"
                "**Remove role** - Take role from a selected member.\n"
                "**Strip ALL roles** - Clean all roles from a user.\n"
                "**Delete role** - Permanently delete a role from the server."
            )
            emb = discord.Embed(title="Role Commands", description=desc, color=discord.Color.blue())
        else:
            desc = (
                "**Lock channel** - Block sending messages in a channel.\n"
                "**Unlock channel** - Restore message permissions.\n"
                "**Set Slowmode** - Set channel cooldown delay.\n"
                "**Delete channel** - Delete channels or categories.\n"
                "**Purge messages** - Quick-delete up to 100 messages.\n"
                "**Create Text Channel** - Create text channel with custom name.\n"
                "**Create Voice Channel** - Create voice channel with custom name.\n"
                "**Create Category** - Create a new category folder."
            )
            emb = discord.Embed(title="Channel and Chat Commands", description=desc, color=discord.Color.green())
            
        await interaction.followup.send(embed=emb, ephemeral=True)


class HelpView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=60)
        self.add_item(HelpDropdown())


@bot.tree.command(name="do", description="Master Control Panel: Server administration via UI elements")
@app_commands.default_permissions(administrator=True)
@app_commands.choices(action=[
    app_commands.Choice(name="Ban member", value="ban"),
    app_commands.Choice(name="Kick member", value="kick"),
    app_commands.Choice(name="Timeout member", value="timeout"),
    app_commands.Choice(name="View member info", value="user_info"),
    app_commands.Choice(name="Add role to member", value="role_add"),
    app_commands.Choice(name="Remove role from member", value="role_remove"),
    app_commands.Choice(name="Strip ALL roles from member", value="strip_roles"),
    app_commands.Choice(name="Delete role from server", value="role_delete"),
    app_commands.Choice(name="Create Text Channel", value="create_text"),
    app_commands.Choice(name="Create Voice Channel", value="create_voice"),
    app_commands.Choice(name="Create Category", value="create_category"),
    app_commands.Choice(name="Lock channel", value="lock_channel"),
    app_commands.Choice(name="Unlock channel", value="unlock_channel"),
    app_commands.Choice(name="Set Slowmode", value="slowmode"),
    app_commands.Choice(name="Delete channel or category", value="delete_channel"),
    app_commands.Choice(name="Purge messages", value="purge")
])
async def do_command(interaction: discord.Interaction, action: app_commands.Choice[str]):
    val = action.value

    if val == "create_text":
        return await interaction.response.send_modal(CreateChannelModal(channel_type="text"))
    elif val == "create_voice":
        return await interaction.response.send_modal(CreateChannelModal(channel_type="voice"))
    elif val == "create_category":
        return await interaction.response.send_modal(CreateCategoryModal())
    elif val == "slowmode":
        return await interaction.response.send_modal(CustomSlowmodeModal())
    elif val == "purge":
        return await interaction.response.send_modal(CustomPurgeModal())
            
    view = DynamicActionView(action=val)
    await interaction.response.send_message("System Panel: Select action below:", view=view, ephemeral=True)

@bot.tree.command(name="help", description="Help Menu: Details regarding all master panel options")
@app_commands.default_permissions(administrator=True)
async def help_command(interaction: discord.Interaction):
    emb = discord.Embed(
        title="Help System - Master Panel",
        description="This bot utilizes the `/do` command to initiate graphic setups for operations.\n\nSelect a category below to see detailed parameters.",
        color=discord.Color.gold()
    )
    view = HelpView()
    await interaction.response.send_message(embed=emb, view=view, ephemeral=True)

DISCORD_TOKEN = "YOUR_DISCORD_BOT_TOKEN"

bot.run('discord token')```
