const { Client, GatewayIntentBits } = require('discord.js');
const fs = require('fs');
const path = require('path');
const tesseract = require('tesseract.js');
const i18next = require('i18next');
const config = require('./config.json');
const { processCode, generateCode, analyzeFile } = require('./commands');

// إعدادات اللغة
i18next.init({
  lng: 'en', // يمكن تغييره إلى 'ar' للغة العربية
  resources: {
    en: require('./languages/english'),
    ar: require('./languages/arabic')
  }
});

const client = new Client({ intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages, GatewayIntentBits.MessageContent] });

client.on('ready', () => {
  console.log(`Logged in as ${client.user.tag}`);
});

client.on('messageCreate', async (message) => {
  if (message.author.bot) return;

  // إذا كان النص يحتوي على "أريد كذا" أو طلب توليد أكواد
  if (message.content.includes("أريد كذا") || message.content.includes("generate")) {
    const generatedCode = await generateCode(message.content);
    message.reply(generatedCode);
  }

  // إذا تم إرسال صورة تحتوي على خطأ
  if (message.attachments.size > 0) {
    const attachment = message.attachments.first();
    if (attachment.contentType.startsWith('image/')) {
      tesseract.recognize(attachment.url, 'eng')
        .then(({ data: { text } }) => {
          const solution = await processCode(text);
          message.reply(solution);
        });
    }
  }

  // إذا كان المستخدم يرسل ملف
  if (message.attachments.size > 0) {
    const file = message.attachments.first();
    if (file.contentType === 'application/javascript') {
      const filePath = path.join(__dirname, file.name);
      await file.download();
      const fileContent = fs.readFileSync(filePath, 'utf-8');
      const solution = await analyzeFile(fileContent);
      message.reply(solution);
    }
  }
});

client.login(config.DISCORD_TOKEN);
