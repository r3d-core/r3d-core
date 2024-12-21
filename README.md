import discord
from discord import app_commands
from discord.ext import commands, tasks
from discord.ui import Button, View, Select
from discord import ButtonStyle, Interaction, Embed, utils
import random
import os
import io
import asyncio
from datetime import timedelta
import datetime
from datetime import datetime
from collections import defaultdict
import time



# Bot setup
intents = discord.Intents.default()
intents.message_content = True
intents.voice_states = True
intents.messages = True
intents.guilds = True
intents.guild_messages = True
intents.members = True
intents.presences = True
bot = commands.Bot(command_prefix="!", intents=intents)

guild_ticket_settings = {}
TicketRenameModal = {}
closed_tickets = {}  
open_tickets = {}

@bot.tree.command(name="ticket_transcript_channel", description="📝 Set the channel where ticket transcripts will be sent.")
@commands.has_permissions(administrator=True)
async def ticket_transcript_channel(interaction: discord.Interaction, channel: discord.TextChannel):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}, "log_channel_id": None}

    guild_ticket_settings[guild_id]["log_channel_id"] = channel.id
    embed = discord.Embed(
        title="📜 Log Channel Set!",
        description=f"The channel for ticket transcripts has been set to: {channel.mention}",
        color=discord.Color.green()
    )
    await interaction.response.send_message(embed=embed)



@bot.tree.command(name="ticket_setup_image", description="🎨 Set the image for the ticket setup message")
@commands.has_permissions(administrator=True)
async def ticket_setup_image(interaction: discord.Interaction, image_url: str):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}}
    
    guild_ticket_settings[guild_id]["setup_image"] = image_url
    embed = discord.Embed(
        title="🖼️ Image Set!",
        description=f"The setup image has been set successfully:\n{image_url}",
        color=discord.Color.green()
    )
    embed.set_thumbnail(url=image_url)
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name="ticket_in_image", description="🎟️ Set the image for inside ticket messages")
@commands.has_permissions(administrator=True)
async def ticket_in_image(interaction: discord.Interaction, image_url: str):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}}

    guild_ticket_settings[guild_id]["in_ticket_image"] = image_url
    embed = discord.Embed(
        title="📸 Inside Ticket Image Set!",
        description=f"The in-ticket image has been updated:\n{image_url}",
        color=discord.Color.green()
    )
    embed.set_thumbnail(url=image_url)
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name="ticket_open_category", description="🏷️ Set the category where tickets should open")
@commands.has_permissions(administrator=True)
async def ticket_open_category(interaction: discord.Interaction, category: discord.CategoryChannel):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}}

    guild_ticket_settings[guild_id]["category_id"] = category.id
    embed = discord.Embed(
        title="📂 Ticket Category Set!",
        description=f"Tickets will now open in the category: **{category.name}**",
        color=discord.Color.blue()
    )
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name="ticket_add_button", description="➕ Add a custom button to the ticket system")
@commands.has_permissions(administrator=True)
async def ticket_add_button(interaction: discord.Interaction, label: str, role: discord.Role):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}}

    button_id = f"ticket_button_{len(guild_ticket_settings[guild_id]['buttons']) + 1}"
    guild_ticket_settings[guild_id]["buttons"][button_id] = {"label": label, "role_id": role.id}
    embed = discord.Embed(
        title="✅ Button Added!",
        description=f"Button **'{label}'** has been added for role: {role.mention}",
        color=discord.Color.green()
    )
    embed.set_footer(text=f"Button ID: {button_id}")
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name="ticket_setup", description="🎫 Set up a ticket system with buttons")
@commands.has_permissions(administrator=True)
async def ticket_setup(interaction: discord.Interaction):
    guild_id = interaction.guild.id
    if guild_id not in guild_ticket_settings:
        guild_ticket_settings[guild_id] = {"setup_image": None, "in_ticket_image": None, "category_id": None, "buttons": {}}

    embed = discord.Embed(
        title="🎟️ Ticket System",
        description="Click one of the buttons below to open a ticket.",
        color=discord.Color.blurple()
    )
    if guild_ticket_settings[guild_id]["setup_image"]:
        embed.set_image(url=guild_ticket_settings[guild_id]["setup_image"])

    view = View()
    for button_id, button_data in guild_ticket_settings[guild_id]["buttons"].items():
        button = Button(label=button_data["label"], style=discord.ButtonStyle.green, custom_id=button_id)
        view.add_item(button)
        
    await interaction.response.send_message(embed=embed, view=view)



@bot.event
async def on_interaction(interaction: Interaction):
    if interaction.type == discord.InteractionType.component:
        custom_id = interaction.data["custom_id"]
        guild_id = interaction.guild.id

        # فتح التذكرة
        if custom_id.startswith("ticket_button_"):
            if guild_id not in guild_ticket_settings or "category_id" not in guild_ticket_settings[guild_id]:
                await interaction.response.send_message("❌ No ticket category set for this server. Set it using /ticket_open_category.", ephemeral=True)
                return

            category_id = guild_ticket_settings[guild_id]["category_id"]
            category = utils.get(interaction.guild.categories, id=category_id)

            if not category:
                await interaction.response.send_message("❌ Category not found. Set it with /ticket_open_category.", ephemeral=True)
                return

            channel_name = f"🎫-{interaction.user.name}"
            channel = await category.create_text_channel(channel_name)
            await channel.set_permissions(interaction.guild.default_role, read_messages=False)
            await channel.set_permissions(interaction.user, read_messages=True, send_messages=True)

            button_data = guild_ticket_settings[guild_id]["buttons"].get(custom_id)
            role_mention = ""
            if button_data:
                role = interaction.guild.get_role(button_data["role_id"])
                if role:
                    await channel.set_permissions(role, read_messages=True, send_messages=True)
                    role_mention = role.mention

            open_tickets[channel.id] = {"owner": interaction.user.id, "claimer": None}

            embed = Embed(
                title="🎟️ Ticket Opened",
                description=f"**[🫅]Ticket Owner:** {interaction.user.mention}\n\n**[🛠️]Support Role:** {role_mention}",
                color=discord.Color.green()
            )
            if guild_ticket_settings[guild_id]["in_ticket_image"]:
                embed.set_image(url=guild_ticket_settings[guild_id]["in_ticket_image"])
                embed.set_thumbnail(url="https://media.wickdev.me/Zo3Ho64wQV.jpg")

            options_view = View()
            options_view.add_item(Button(label="Ticket Options ⚙️", style=ButtonStyle.success, custom_id="ticket_options"))

            await channel.send(f"  {interaction.user.mention} | {role_mention} ")
            await channel.send(embed=embed, view=options_view)
            await interaction.response.send_message(f"✅ Ticket opened: {channel.mention}", ephemeral=True)

        # قائمة الخيارات
        elif custom_id == "ticket_options":
            options_view = View()
            options_view.add_item(Button(label="Claim Ticket ✅", style=ButtonStyle.blurple, custom_id="ticket_claim"))
            options_view.add_item(Button(label="Close Ticket 🔒", style=ButtonStyle.red, custom_id="ticket_close"))
            options_view.add_item(Button(label="Call Staff 📞", style=ButtonStyle.green, custom_id="ticket_call_staff"))
            options_view.add_item(Button(label="Rename Ticket ✏️", style=ButtonStyle.gray, custom_id="ticket_rename"))
            options_view.add_item(Button(label="Transfer Ticket 🔄", style=ButtonStyle.primary, custom_id="ticket_transfer"))

            await interaction.response.send_message("Choose an action:", view=options_view, ephemeral=True)

        # استلام التذكرة
        elif custom_id == "ticket_claim":
            channel = interaction.channel
            if channel.id not in open_tickets:
                await interaction.response.send_message("❌ This is not an open ticket.", ephemeral=True)
                return
            if open_tickets[channel.id]["claimer"]:
                await interaction.response.send_message("❌ This ticket is already claimed.", ephemeral=True)
                return

            open_tickets[channel.id]["claimer"] = interaction.user.id
            embed = Embed(
                title="✅ Ticket Claimed",
                description=f"{interaction.user.mention} has claimed this ticket.",
                color=discord.Color.green()
            )
            await channel.send(embed=embed)
            await interaction.response.send_message("✅ Ticket claimed successfully.", ephemeral=True)

        # إغلاق التذكرة
        if custom_id == "ticket_close":
            channel = interaction.channel
            if channel.id not in open_tickets:
                await interaction.response.send_message("❌ This is not an open ticket.", ephemeral=True)
                return
            
            # جمع الرسائل من القناة الخاصة بالتذكرة (الـ transcript)
            messages = []
            async for message in channel.history(limit=100):  # يمكنك تعديل limit لعدد أكبر من الرسائل إذا كنت بحاجة
                messages.append(f"**{message.author}:** {message.content}")

            # إرسال السجل إلى قناة اللوج
            if guild_id in guild_ticket_settings and "log_channel_id" in guild_ticket_settings[guild_id]:
                log_channel_id = guild_ticket_settings[guild_id]["log_channel_id"]
                log_channel = interaction.guild.get_channel(log_channel_id)

                if log_channel:
                    # تجميع جميع الرسائل المجمعة في نص واحد
                    transcript_text = "\n".join(messages)

                    embed = discord.Embed(
                        title="🔒 Ticket Closed - Transcript",
                        description=f"**Ticket Closed By:** {interaction.user.mention}\n**Ticket Owner:** <@{open_tickets[channel.id]['owner']}>\n**Ticket Channel:** {channel.mention}",
                        color=discord.Color.red()
                    )
                    embed.set_footer(text=f"Ticket Closed")
                    embed.add_field(name="Transcript:", value=f"```{transcript_text}```", inline=False)

                    await log_channel.send(embed=embed)

            # إغلاق القناة بعد إرسال السجل
            await channel.delete()
            await interaction.response.send_message("✅ Ticket closed and transcript sent.", ephemeral=True)
            
            # إرسال السجل إلى قناة اللوج
            if guild_id in guild_ticket_settings and "log_channel_id" in guild_ticket_settings[guild_id]:
                log_channel_id = guild_ticket_settings[guild_id]["log_channel_id"]
                log_channel = interaction.guild.get_channel(log_channel_id)

                if log_channel:
                    embed = discord.Embed(
                        title="🔒 Ticket Closed",
                        description=f"**Ticket Closed By:** {interaction.user.mention}\n**Ticket Owner:** <@{open_tickets[channel.id]['owner']}>\n**Ticket Channel:** {channel.mention}",
                        color=discord.Color.red()
                    )
                    embed.set_footer(text=f"Closed at {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
                    await log_channel.send(embed=embed)

            await channel.delete()
            await interaction.response.send_message("✅ Ticket closed.", ephemeral=True)

        # طلب الدعم
        elif custom_id == "ticket_call_staff":
            channel = interaction.channel
            role = discord.utils.get(interaction.guild.roles, name="Support")  # Adjust role name
            if role:
                await channel.send(f"🚨 Support staff, please assist {interaction.user.mention}! {role.mention}")
            await interaction.response.send_message(f"**🚨|| @everyone || Staff Requested **")

        # إعادة تسمية التذكرة
        elif custom_id == "ticket_rename":
            await interaction.response.send_modal(TicketRenameModal())

        # نقل التذكرة
        elif custom_id == "ticket_transfer":
            await interaction.response.send_modal(TicketTransferModal())

class TicketTransferModal(discord.ui.Modal, title="Transfer Ticket"):
    new_role_id = discord.ui.TextInput(label="Enter the ID of the new role", placeholder="Enter the role ID to transfer the ticket to")

    async def on_submit(self, interaction: discord.Interaction):
        # Get the channel and new role ID from the user input
        channel = interaction.channel
        new_role_id = self.new_role_id.value

        try:
            new_role = discord.utils.get(interaction.guild.roles, id=int(new_role_id))
        except ValueError:
            await interaction.response.send_message("❌ Invalid role ID. Please enter a valid number for the role ID.", ephemeral=True)
            return

        if not new_role:
            await interaction.response.send_message("❌ Role not found. Please make sure the role ID is correct.", ephemeral=True)
            return

        # Get the original user (ticket owner)
        owner = interaction.guild.get_member(open_tickets[channel.id]["owner"])

        # 1. Remove all permissions from everyone (disable message sending for everyone)
        await channel.set_permissions(interaction.guild.default_role, send_messages=False, read_messages=False)

        # 2. Remove permissions from all roles in the channel
        for role in interaction.guild.roles:
            if role != new_role and role != interaction.guild.default_role:  # Skip the new role and @everyone
                await channel.set_permissions(role, send_messages=False, read_messages=False)

        # 3. Grant "send messages" permission for the new role (the role to which the ticket is being transferred)
        await channel.set_permissions(new_role, read_messages=True, send_messages=True)

        # 4. Grant "send messages" permission for the ticket owner (the user who created the ticket)
        if owner:
            await channel.set_permissions(owner, read_messages=True, send_messages=True)

        # Update ticket's embed and let the user know
        embed = discord.Embed(
            title="✅ Ticket Transferred",
            description=f"The ticket has been successfully transferred to the role: {new_role.name} (ID: {new_role.id}).",
            color=discord.Color.green()
        )

        await channel.send(embed=embed)
        await interaction.response.send_message(f"✅ The ticket has been transferred to the role: {new_role.name} (ID: {new_role.id}).", ephemeral=True)


bot.run("MTMxOTYzMDQ3ODAwNDcxNTU4Mg.GqhTJ4.Jr0zH4rdQqiOPp3X8jjhNXPuoyqdHpHcvj_MB0")

<!---
r3d-core/r3d-core is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
