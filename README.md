const {
  Client,
  GatewayIntentBits,
  Partials,
  EmbedBuilder,
  PermissionsBitField,
} = require("discord.js");

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers,
  ],
  partials: [Partials.Channel],
});

const mediaChannels = new Set();
const guildLogChannels = new Map(); // serverID -> logChannelID

client.once("ready", () => {
  console.log(`Logged in as ${client.user.tag}`);
});

const createModEmbed = (action, targetUser, moderator, reason) => {
  return new EmbedBuilder()
    .setTitle(`Moderation | ${action}`)
    .setColor(0xff5733)
    .addFields(
      { name: "User", value: `${targetUser.user.tag || targetUser.tag}`, inline: true },
      { name: "Moderator", value: `${moderator.tag}`, inline: true },
      { name: "Reason", value: reason },
    )
    .setTimestamp();
};

const sendLogs = async (guild, embed) => {
  const logChannelId = guildLogChannels.get(guild.id);
  let logChannel = logChannelId
    ? guild.channels.cache.get(logChannelId)
    : guild.channels.cache.find((c) => c.name === "mod-logs");

  if (!logChannel) {
    logChannel = await guild.channels.create({
      name: "mod-logs",
      type: 0,
    });
  }

  logChannel.send({ embeds: [embed] }).catch(() => {});
};

client.on("messageCreate", async (message) => {
  if (!message.content.startsWith("!") || message.author.bot) return;

  const args = message.content.slice(1).trim().split(/ +/);
  const command = args.shift().toLowerCase();
  const target = message.mentions.members?.first();
  const reason = args.slice(1).join(" ") || "No reason provided";

  switch (command) {
    case "avatar": {
      if (!target) return message.reply("Mention a user!");
      const embed = new EmbedBuilder()
        .setTitle(`${target.user.username}'s Avatar`)
        .setImage(target.user.displayAvatarURL({ dynamic: true, size: 512 }))
        .setColor(0x3498db);
      message.channel.send({ embeds: [embed] });
      break;
    }

    case "timeout": {
      if (!target) return message.reply("Mention a user to timeout!");
      const duration = parseInt(args[1]) * 1000; // in ms
      if (isNaN(duration) || duration <= 0)
        return message.reply("Provide timeout duration in seconds!");

      await target.timeout(duration, reason).catch(() => {});
      const embed = createModEmbed("Timeout", target, message.author, reason);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "removetimeout": {
      if (!target) return message.reply("Mention a user to remove timeout!");
      await target.timeout(null).catch(() => {});
      const embed = createModEmbed("Remove Timeout", target, message.author, reason);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "kick": {
      if (!target) return message.reply("Mention a user to kick!");
      await target.kick(reason).catch(() => {});
      const embed = createModEmbed("Kick", target, message.author, reason);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "ban": {
      if (!target) return message.reply("Mention a user to ban!");
      await target.ban({ reason }).catch(() => {});
      const embed = createModEmbed("Ban", target, message.author, reason);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "unban": {
      const userId = args[0];
      if (!userId) return message.reply("Provide a user ID to unban!");

      await message.guild.bans.remove(userId, reason).catch(() => {});
      const fakeTarget = { tag: `UserID: ${userId}` };
      const embed = createModEmbed("Unban", fakeTarget, message.author, reason);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "say": {
      const text = args.join(" ");
      if (!text) return message.reply("Provide a message to send.");
      message.delete().catch(() => {});
      message.channel.send(text);
      break;
    }

    case "giverole": {
      const role = message.mentions.roles.first();
      if (!target || !role) return message.reply("Mention a user and a role!");
      await target.roles.add(role).catch(() => {});
      const embed = createModEmbed("Give Role", target, message.author, `Role: ${role.name}`);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "removerole": {
      const role = message.mentions.roles.first();
      if (!target || !role) return message.reply("Mention a user and a role!");
      await target.roles.remove(role).catch(() => {});
      const embed = createModEmbed("Remove Role", target, message.author, `Role: ${role.name}`);
      message.channel.send({ embeds: [embed] });
      sendLogs(message.guild, embed);
      break;
    }

    case "setmedia": {
      mediaChannels.add(message.channel.id);
      message.reply(`This channel is now media-only.`);
      break;
    }

    case "clear": {
      const count = parseInt(args[0], 10);
      if (!count || count < 1 || count > 100)
        return message.reply("Enter a number between 1 and 100.");
      await message.channel.bulkDelete(count, true).catch(() => {});
      message.channel
        .send(`Cleared ${count} messages.`)
        .then((m) => setTimeout(() => m.delete(), 3000));
      break;
    }

    case "setlog": {
      guildLogChannels.set(message.guild.id, message.channel.id);
      message.reply(`Set ${message.channel} as the moderation log channel.`);
      break;
    }

    case "lock": {
      if (!message.member.permissions.has(PermissionsBitField.Flags.MANAGE_CHANNELS)) return message.reply("You don't have permission to lock this channel.");
      await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, { SendMessages: false });
      message.reply("Channel locked.");
      break;
    }

    case "unlock": {
      if (!message.member.permissions.has(PermissionsBitField.Flags.MANAGE_CHANNELS)) return message.reply("You don't have permission to unlock this channel.");
      await message.channel.permissionOverwrites.edit(message.guild.roles.everyone, { SendMessages: true });
      message.reply("Channel unlocked.");
      break;
    }

    case "serverinfo": {
      const serverEmbed = new EmbedBuilder()
        .setTitle(`${message.guild.name} Server Info`)
        .setColor(0x00bfff)
        .addFields(
          { name: "Server Name", value: message.guild.name },
          { name: "Total Members", value: message.guild.memberCount.toString() },
          { name: "Created On", value: message.guild.createdAt.toDateString() },
          { name: "Region", value: message.guild.preferredLocale },
        );
      message.channel.send({ embeds: [serverEmbed] });
      break;
    }

    case "help": {
      const helpEmbed = new EmbedBuilder()
        .setTitle("Moderation Bot Commands")
        .setColor(0x00bfff)
        .setDescription("Prefix: `!`")
        .addFields(
          { name: "**User Commands**", value: "`!avatar @user`, `!say message`" },
          { name: "**Moderation Commands**", value: "`!timeout @user seconds reason`, `!removetimeout @user`, `!kick @user reason`, `!ban @user reason`, `!unban userID reason`" },
          { name: "**Role Commands**", value: "`!giverole @user @role`, `!removerole @user @role`" },
          { name: "**Channel Commands**", value: "`!setmedia`, `!clear number`, `!setlog`, `!lock`, `!unlock`" },
          { name: "**Server Info**", value: "`!serverinfo`" },
          { name: "**Info**", value: "`!help` (this menu)" }
        )
        .setFooter({ text: "Bot by You" });
      message.channel.send({ embeds: [helpEmbed] });
      break;
    }
  }
});

client.on("messageCreate", (message) => {
  if (
    mediaChannels.has(message.channel.id) &&
    !message.attachments.size &&
    !message.author.bot
  ) {
    message.delete().catch(() => {});
    message.channel
      .send(`${message.author}, only media is allowed here.`)
      .then((m) => setTimeout(() => m.delete(), 5000));
  }
});

client.login("");
