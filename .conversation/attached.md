import express from "express";
import admin from "firebase-admin";
import { GoogleGenAI } from "@google/genai";
import {
  Client,
  Events,
  GatewayIntentBits,
  PermissionFlagsBits,
  REST,
  Routes,
  SlashCommandBuilder,
  MessageFlags,
} from "discord.js";

const token = process.env.DISCORD_TOKEN;
const clientId = process.env.DISCORD_CLIENT_ID;
const port = Number(process.env.PORT ?? 3000);

const levelUpChannelId =
  process.env.LEVEL_UP_CHANNEL_ID;

const generalChannelId =
  process.env.GENERAL_CHANNEL_ID;

/*
====================================================
👑 BOT OWNER
====================================================

ضع في Render:

OWNER_ID=1521900880222490743

هذا الحساب فقط يستطيع التحكم بإعدادات البوت.
*/

const OWNER_ID =
  process.env.OWNER_ID ||
  "1521900880222490743";

// ========================================
// 🔑 AI API Keys
// ========================================

const geminiApiKey =
  process.env.GEMINI_API_KEY;

const groqApiKey =
  process.env.GROQ_API_KEY;

const openRouterApiKey =
  process.env.OPENROUTER_API_KEY;

// ========================================
// 🔐 Basic Configuration
// ========================================

if (
  !token ||
  !clientId ||
  !levelUpChannelId
) {
  throw new Error(
    "Missing bot configuration. Add DISCORD_TOKEN, DISCORD_CLIENT_ID, and LEVEL_UP_CHANNEL_ID.",
  );
}

if (
  !process.env.FIREBASE_DATABASE_URL ||
  !process.env.FIREBASE_PRIVATE_KEY ||
  !process.env.FIREBASE_CLIENT_EMAIL ||
  !process.env.FIREBASE_PROJECT_ID
) {
  throw new Error(
    "Missing Firebase environment variables.",
  );
}

if (
  !geminiApiKey &&
  !groqApiKey &&
  !openRouterApiKey
) {
  throw new Error(
    "Missing AI API keys. Add at least one of GEMINI_API_KEY, GROQ_API_KEY, or OPENROUTER_API_KEY.",
  );
}

// ========================================
// 🌐 Firebase
// ========================================

admin.initializeApp({
  credential: admin.credential.cert({
    projectId:
      process.env.FIREBASE_PROJECT_ID,

    clientEmail:
      process.env.FIREBASE_CLIENT_EMAIL,

    privateKey:
      process.env.FIREBASE_PRIVATE_KEY.replace(
        /\\n/g,
        "\n",
      ),
  }),

  databaseURL:
    process.env.FIREBASE_DATABASE_URL,
});

const db = admin.database();

// ========================================
// 🧠 Gemini
// ========================================

const genAI = geminiApiKey
  ? new GoogleGenAI({
      apiKey: geminiApiKey,
    })
  : null;

const GEMINI_MODEL =
  "gemini-2.5-flash";

// ========================================
// ⚡ Groq
// ========================================

const GROQ_MODEL =
  process.env.GROQ_MODEL ||
  "llama-3.1-8b-instant";

// ========================================
// 🌐 OpenRouter
// ========================================

const OPENROUTER_MODEL =
  process.env.OPENROUTER_MODEL ||
  "openrouter/free";

// ========================================
// 🧠 AI System Instruction
// ========================================

const AI_SYSTEM_INSTRUCTION =
  "أنت مساعد ذكي ولطيف لسيرفر ديسكورد عربي فخم اسمه 3RB. " +
  "أجب بالعربية بشكل طبيعي وودود ومختصر. " +
  "استخدم اللهجة العربية المناسبة للسياق بدون مبالغة. " +
  "لا تذكر أنك نموذج ذكاء اصطناعي إلا إذا سألك العضو عن ذلك. " +
  "لا تخترع معلومات مؤكدة عن السيرفر إذا لم تكن موجودة في السؤال. " +
  "لا تدعي أنك تستطيع تنفيذ أوامر إدارية أو تغيير إعدادات السيرفر بنفسك. " +
  "لا تطلب من الأعضاء أسرار البوت أو مفاتيح API أو كلمات المرور.";

// ========================================
// 💬 Fallback Replies
// ========================================

const fallbackReplies = [
  "هلا بك! المعالج مشغول حبتين حالياً، اطلبني بعد ثواني وبخدمك 🤍",

  "موجود معك! بس الذكاء الاصطناعي مأخذ استراحة بسيطة ☕",

  "سم يا غالي، أسمعك بس الشات عليه ضغط، جرب تسألني مرة ثانية!",

  "أرحب! جاهز لأي خدمة بالسيرفر، أمرني 🔥",
];

// ========================================
// 🌐 Render Web Server
// ========================================

const app = express();

app.get("/", (_request, response) => {
  response
    .status(200)
    .send(
      "3RB Bot is Online with Firebase & AI Fallback System!",
    );
});

const webServer = app.listen(
  port,
  "0.0.0.0",
  () => {
    console.info(
      `Uptime server listening on port ${port}`,
    );
  },
);

// ========================================
// ⚙️ Discord Commands
// ========================================

const commands = [
  // ======================================
  // /ping
  // ======================================

  new SlashCommandBuilder()
    .setName("ping")
    .setDescription(
      "Check whether the bot is online.",
    ),

  // ======================================
  // /settings
  // ======================================

  new SlashCommandBuilder()
    .setName("settings")
    .setDescription(
      "التحكم الكامل بإعدادات البوت - للمالك فقط.",
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("show")
        .setDescription(
          "عرض إعدادات البوت الحالية.",
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("anti-links")
        .setDescription(
          "تشغيل أو إيقاف حماية الروابط.",
        )
        .addBooleanOption((option) =>
          option
            .setName("enabled")
            .setDescription(
              "تشغيل أو إيقاف.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("auto-salam")
        .setDescription(
          "تشغيل أو إيقاف رد السلام.",
        )
        .addBooleanOption((option) =>
          option
            .setName("enabled")
            .setDescription(
              "تشغيل أو إيقاف.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("faq")
        .setDescription(
          "تشغيل أو إيقاف نظام FAQ.",
        )
        .addBooleanOption((option) =>
          option
            .setName("enabled")
            .setDescription(
              "تشغيل أو إيقاف.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("ai")
        .setDescription(
          "تشغيل أو إيقاف الذكاء الاصطناعي.",
        )
        .addBooleanOption((option) =>
          option
            .setName("enabled")
            .setDescription(
              "تشغيل أو إيقاف.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("hourly")
        .setDescription(
          "تشغيل أو إيقاف الرسائل التلقائية.",
        )
        .addBooleanOption((option) =>
          option
            .setName("enabled")
            .setDescription(
              "تشغيل أو إيقاف.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("general")
        .setDescription(
          "تحديد قناة الشات العام.",
        )
        .addChannelOption((option) =>
          option
            .setName("channel")
            .setDescription(
              "القناة العامة.",
            )
            .setRequired(true),
        ),
    ),

  // ======================================
  // /exempt-channel
  // ======================================

  new SlashCommandBuilder()
    .setName("exempt-channel")
    .setDescription(
      "استثناء قناة من حماية الروابط.",
    )
    .addChannelOption((option) =>
      option
        .setName("channel")
        .setDescription(
          "القناة المستثناة.",
        )
        .setRequired(true),
    ),

  // ======================================
  // /faq
  // ======================================

  new SlashCommandBuilder()
    .setName("faq")
    .setDescription(
      "إدارة الردود التلقائية.",
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("add")
        .setDescription(
          "إضافة أو تحديث رد.",
        )
        .addStringOption((option) =>
          option
            .setName("keyword")
            .setDescription(
              "الكلمة المفتاحية.",
            )
            .setRequired(true),
        )
        .addStringOption((option) =>
          option
            .setName("answer")
            .setDescription(
              "الرد.",
            )
            .setRequired(true),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("delete")
        .setDescription(
          "حذف رد.",
        )
        .addStringOption((option) =>
          option
            .setName("keyword")
            .setDescription(
              "الكلمة المفتاحية.",
            )
            .setRequired(true),
        ),
    ),

  // ======================================
  // /banword
  // ======================================

  new SlashCommandBuilder()
    .setName("banword")
    .setDescription(
      "إدارة الكلمات الممنوعة.",
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("add")
        .setDescription(
          "إضافة كلمة ممنوعة.",
        )
        .addStringOption((option) =>
          option
            .setName("keyword")
            .setDescription(
              "الكلمة.",
            )
            .setRequired(true),
        )
        .addIntegerOption((option) =>
          option
            .setName("duration")
            .setDescription(
              "مدة الكتم بالدقائق.",
            )
            .setRequired(true)
            .setMinValue(1)
            .setMaxValue(40320),
        ),
    )

    .addSubcommand((subcommand) =>
      subcommand
        .setName("delete")
        .setDescription(
          "حذف كلمة ممنوعة.",
        )
        .addStringOption((option) =>
          option
            .setName("keyword")
            .setDescription(
              "الكلمة.",
            )
            .setRequired(true),
        ),
    ),
].map((command) =>
  command.toJSON(),
);

// ========================================
// 🤖 Discord Client
// ========================================

const rest = new REST({
  version: "10",
}).setToken(token);

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ],
});

// ========================================
// 🧠 Memory / Cooldowns
// ========================================

const salamCooldowns = new Map();
const processedMessages = new Set();
const xpCooldowns = new Map();
const aiCooldowns = new Map();

// ========================================
// ⏰ Hourly
// ========================================

const hourlyReminders = [
  "⏰ **رسالة تلقائية:** سبحان الله وبحمده، سبحان الله العظيم 🤍",

  "⏰ **رسالة تلقائية:** لا إله إلا أنت سبحانك إني كنت من الظالمين 🌿",

  "⏰ **رسالة تلقائية:** استغفر الله العظيم وأتوب إليه 🤍",

  "⏰ **رسالة تلقائية:** لا حول ولا قوة إلا بالله العلي العظيم 🌸",

  "⏰ **رسالة تلقائية:** الحمد لله حمداً كثيراً طيباً مباركاً فيه ⭐",

  "⏰ **رسالة تلقائية:** اللهم صل وسلم على نبينا محمد وعلى آله وصحبه أجمعين 🤲",

  "⏰ **رسالة تلقائية:** سبحان الله، والحمد لله، ولا إله إلا الله، والله أكبر 💎",
];

// ========================================
// 🔐 OWNER CHECK
// ========================================

function isBotOwner(userId) {
  return String(userId) ===
    String(OWNER_ID);
}

// ========================================
// 🔐 Owner Only Reply
// ========================================

async function requireOwner(interaction) {
  if (
    isBotOwner(interaction.user.id)
  ) {
    return true;
  }

  const response = {
    content:
      "⛔ هذا الأمر مخصص لمالك البوت فقط.",
    flags: MessageFlags.Ephemeral,
  };

  if (
    interaction.replied ||
    interaction.deferred
  ) {
    await interaction
      .followUp(response)
      .catch(() => {});
  } else {
    await interaction
      .reply(response)
      .catch(() => {});
  }

  return false;
}

// ========================================
// 🌐 Firebase Settings
// ========================================

async function getSettings(guildId) {
  const snapshot = await db
    .ref(`settings/${guildId}`)
    .once("value");

  if (!snapshot.exists()) {
    const defaultSettings = {
      guild_id: guildId,

      general_channel:
        generalChannelId || null,

      anti_link_enabled: 1,

      auto_salam_enabled: 1,

      faq_enabled: 1,

      ai_enabled: 1,

      hourly_enabled: 1,

      exempt_channels: "",
    };

    await db
      .ref(`settings/${guildId}`)
      .set(defaultSettings);

    return defaultSettings;
  }

  const settings = snapshot.val();

  // تحديث الإعدادات القديمة إذا لم تكن تحتوي على الحقول الجديدة

  if (
    settings.ai_enabled === undefined
  ) {
    settings.ai_enabled = 1;
  }

  if (
    settings.hourly_enabled ===
    undefined
  ) {
    settings.hourly_enabled = 1;
  }

  return settings;
}

// ========================================
// 💬 FAQ
// ========================================

async function getFaqs(guildId) {
  const snapshot = await db
    .ref(`faqs/${guildId}`)
    .once("value");

  return snapshot.exists()
    ? snapshot.val()
    : {};
}

// ========================================
// 🔇 Ban Words
// ========================================

async function getBanWords(guildId) {
  const snapshot = await db
    .ref(`banwords/${guildId}`)
    .once("value");

  return snapshot.exists()
    ? snapshot.val()
    : {};
}

// ========================================
// 🛠️ Helpers
// ========================================

function exemptChannelIds(value) {
  return (value ?? "")
    .split(",")
    .map((channelId) =>
      channelId.trim(),
    )
    .filter(Boolean);
}

function isSalam(message) {
  return /(?:السلام|سلام)\s*[عع]ليكم|السَّلامُ?\s*عليكم/i.test(
    message
      .trim()
      .replace(
        /[،.!؟?]/g,
        "",
      ),
  );
}

function containsLinkOrInvite(content) {
  return /https?:\/\/\S+|(?:discord\.gg|discord\.com\/invite)\/\S+/i.test(
    content,
  );
}

// ========================================
// ⭐ Level Detection
// ========================================

function isLevelQuestion(content) {
  const text = content
    .toLocaleLowerCase()
    .replace(
      /[؟?!.,،]/g,
      "",
    )
    .trim();

  return (
    text.includes("كم لفلي") ||
    text.includes("كم لفل") ||
    text.includes("كم مستواي") ||
    text.includes("ما هو مستواي") ||
    text.includes("وش لفلي") ||
    text.includes("وش مستواي") ||
    text.includes("لفلي كم") ||
    text.includes("مستواي كم")
  );
}

// ========================================
// 🤖 AI Error
// ========================================

function getErrorStatus(error) {
  if (!error) return null;

  if (
    typeof error.status ===
    "number"
  ) {
    return error.status;
  }

  if (
    typeof error.statusCode ===
    "number"
  ) {
    return error.statusCode;
  }

  const message = String(
    error?.message || error,
  );

  const match =
    message.match(
      /\b(400|401|403|404|408|409|429|500|502|503|504)\b/,
    );

  return match
    ? Number(match[1])
    : null;
}

function isFallbackError(error) {
  const status =
    getErrorStatus(error);

  const errorText = String(
    error?.message || error,
  ).toUpperCase();

  return (
    status === 400 ||
    status === 401 ||
    status === 403 ||
    status === 404 ||
    status === 408 ||
    status === 409 ||
    status === 429 ||
    status === 500 ||
    status === 502 ||
    status === 503 ||
    status === 504 ||
    errorText.includes(
      "RESOURCE_EXHAUSTED",
    ) ||
    errorText.includes(
      "RATE LIMIT",
    ) ||
    errorText.includes(
      "TOO MANY REQUESTS",
    ) ||
    errorText.includes(
      "UNAVAILABLE",
    ) ||
    errorText.includes(
      "TIMEOUT",
    ) ||
    errorText.includes(
      "DEADLINE_EXCEEDED",
    ) ||
    errorText.includes(
      "INTERNAL SERVER ERROR",
    ) ||
    errorText.includes(
      "MODEL IS UNAVAILABLE",
    ) ||
    errorText.includes(
      "DOES NOT EXIST",
    ) ||
    errorText.includes(
      "DO NOT HAVE ACCESS",
    ) ||
    errorText.includes(
      "NOT FOUND",
    ) ||
    errorText.includes(
      "FORBIDDEN",
    ) ||
    errorText.includes(
      "UNAUTHORIZED",
    ) ||
    errorText.includes(
      "FREE",
    )
  );
}

// ========================================
// ⏳ Sleep
// ========================================

function sleep(ms) {
  return new Promise(
    (resolve) =>
      setTimeout(resolve, ms),
  );
}

// ========================================
// ⏱️ Fetch Timeout
// ========================================

async function fetchWithTimeout(
  url,
  options = {},
  timeoutMs = 30000,
) {
  const controller =
    new AbortController();

  const timeout = setTimeout(
    () => {
      controller.abort();
    },
    timeoutMs,
  );

  try {
    return await fetch(url, {
      ...options,
      signal:
        controller.signal,
    });
  } catch (error) {
    if (
      error?.name ===
      "AbortError"
    ) {
      const timeoutError =
        new Error(
          `Request timed out after ${timeoutMs}ms.`,
        );

      timeoutError.status = 408;

      throw timeoutError;
    }

    throw error;
  } finally {
    clearTimeout(timeout);
  }
}

// ========================================
// 🥇 Groq
// ========================================

async function askGroq(prompt) {
  if (!groqApiKey) {
    throw new Error(
      "GROQ_API_KEY is not configured.",
    );
  }

  const response =
    await fetchWithTimeout(
      "https://api.groq.com/openai/v1/chat/completions",
      {
        method: "POST",

        headers: {
          "Content-Type":
            "application/json",

          Authorization:
            `Bearer ${groqApiKey}`,
        },

        body: JSON.stringify({
          model: GROQ_MODEL,

          messages: [
            {
              role: "system",
              content:
                AI_SYSTEM_INSTRUCTION,
            },

            {
              role: "user",
              content: prompt,
            },
          ],

          temperature: 0.7,

          max_tokens: 700,
        }),
      },
      30000,
    );

  const data =
    await response
      .json()
      .catch(() => ({}));

  if (!response.ok) {
    const error = new Error(
      data?.error?.message ||
        `Groq API Error ${response.status}`,
    );

    error.status =
      response.status;

    throw error;
  }

  const text =
    data?.choices?.[0]?.message?.content?.trim();

  if (!text) {
    throw new Error(
      "Empty response from Groq.",
    );
  }

  return text;
}

// ========================================
// 🥈 Gemini
// ========================================

async function askGemini(prompt) {
  if (!genAI) {
    throw new Error(
      "GEMINI_API_KEY is not configured.",
    );
  }

  const maxRetries = 3;

  for (
    let attempt = 0;
    attempt < maxRetries;
    attempt++
  ) {
    try {
      const response =
        await genAI.models.generateContent(
          {
            model:
              GEMINI_MODEL,

            contents: prompt,

            config: {
              systemInstruction:
                AI_SYSTEM_INSTRUCTION,

              httpOptions: {
                timeout: 30000,
              },
            },
          },
        );

      const text =
        response.text?.trim();

      if (text) {
        return text;
      }

      throw new Error(
        "Empty response from Gemini API",
      );
    } catch (error) {
      console.error(
        `❌ Gemini Error - Attempt ${
          attempt + 1
        }/${maxRetries}:`,
        error,
      );

      const errorText =
        String(
          error?.message ||
            error,
        );

      const temporary =
        errorText.includes("400") ||
        errorText.includes("401") ||
        errorText.includes("403") ||
        errorText.includes("404") ||
        errorText.includes("408") ||
        errorText.includes("409") ||
        errorText.includes("429") ||
        errorText.includes(
          "RESOURCE_EXHAUSTED",
        ) ||
        errorText.includes("503") ||
        errorText.includes(
          "UNAVAILABLE",
        ) ||
        errorText.includes("500") ||
        errorText.includes(
          "INTERNAL",
        ) ||
        errorText.includes(
          "TIMEOUT",
        ) ||
        errorText.includes(
          "DEADLINE_EXCEEDED",
        );

      if (
        !temporary ||
        attempt ===
          maxRetries - 1
      ) {
        throw error;
      }

      const delay =
        1500 *
        Math.pow(
          2,
          attempt,
        );

      await sleep(delay);
    }
  }

  throw new Error(
    "Gemini failed after all retries.",
  );
}

// ========================================
// 🥉 OpenRouter
// ========================================

async function askOpenRouter(
  prompt,
) {
  if (!openRouterApiKey) {
    throw new Error(
      "OPENROUTER_API_KEY is not configured.",
    );
  }

  const response =
    await fetchWithTimeout(
      "https://openrouter.ai/api/v1/chat/completions",
      {
        method: "POST",

        headers: {
          "Content-Type":
            "application/json",

          Authorization:
            `Bearer ${openRouterApiKey}`,

          "HTTP-Referer":
            "https://discord.com/",

          "X-Title":
            "3RB Discord Bot",
        },

        body: JSON.stringify({
          model:
            OPENROUTER_MODEL,

          messages: [
            {
              role: "system",
              content:
                AI_SYSTEM_INSTRUCTION,
            },

            {
              role: "user",
              content: prompt,
            },
          ],

          temperature: 0.7,

          max_tokens: 700,
        }),
      },
      30000,
    );

  const data =
    await response
      .json()
      .catch(() => ({}));

  if (!response.ok) {
    const error = new Error(
      data?.error?.message ||
        `OpenRouter API Error ${response.status}`,
    );

    error.status =
      response.status;

    throw error;
  }

  const text =
    data?.choices?.[0]?.message?.content?.trim();

  if (!text) {
    throw new Error(
      "Empty response from OpenRouter.",
    );
  }

  return text;
}

// ========================================
// 🧠 AI Fallback
// ========================================

async function askAI(prompt) {
  const providers = [
    {
      name: "Gemini",

      enabled:
        Boolean(geminiApiKey),

      ask: askGemini,
    },

    {
      name:
        "OpenRouter Free",

      enabled:
        Boolean(
          openRouterApiKey,
        ),

      ask: askOpenRouter,
    },

    {
      name: "Groq",

      enabled:
        Boolean(groqApiKey),

      ask: askGroq,
    },
  ];

  const enabledProviders =
    providers.filter(
      (provider) =>
        provider.enabled,
    );

  if (
    enabledProviders.length ===
    0
  ) {
    throw new Error(
      "No AI providers configured.",
    );
  }

  let lastError = null;

  for (const provider of
    enabledProviders) {
    try {
      console.log(
        `🤖 Trying AI provider: ${provider.name}`,
      );

      const result =
        await provider.ask(
          prompt,
        );

      console.log(
        `✅ AI response from ${provider.name}`,
      );

      return result;
    } catch (error) {
      lastError = error;

      console.error(
        `❌ ${provider.name} failed:`,
        error,
      );

      continue;
    }
  }

  throw (
    lastError ||
    new Error(
      "All AI providers failed.",
    )
  );
}

// ========================================
// 📌 Register Commands
// ========================================

async function registerCommands() {
  await rest.put(
    Routes.applicationCommands(
      clientId,
    ),
    {
      body: commands,
    },
  );

  console.info(
    "✅ Slash commands registered.",
  );
}

// ========================================
// ⚠️ Warning
// ========================================

async function sendTemporaryWarning(
  channel,
) {
  const warning =
    await channel.send(
      "⚠️ الروابط والدعوات غير مسموحة هنا. سيتم حذف هذه الرسالة خلال 5 ثوانٍ.",
    );

  setTimeout(() => {
    warning
      .delete()
      .catch(() => {});
  }, 5000);
}

// ========================================
// 🟢 Ready
// ========================================

client.once(
  Events.ClientReady,
  (readyClient) => {
    console.info(
      `Ready as ${readyClient.user.tag}`,
    );

    console.info(
      `👑 Bot Owner ID: ${OWNER_ID}`,
    );

    setInterval(
      async () => {
        try {
          if (!generalChannelId)
            return;

          const channel =
            await client.channels
              .fetch(
                generalChannelId,
              )
              .catch(
                () => null,
              );

          if (
            channel?.isTextBased()
          ) {
            const settings =
              await getSettings(
                channel.guildId,
              );

            if (
              !settings.hourly_enabled
            ) {
              return;
            }

            const randomReminder =
              hourlyReminders[
                Math.floor(
                  Math.random() *
                    hourlyReminders.length,
                )
              ];

            await channel.send(
              randomReminder,
            );
          }
        } catch (err) {
          console.error(
            "Failed to send hourly automated message:",
            err,
          );
        }
      },
      3600000,
    );
  },
);

// ========================================
// ⚙️ Interactions
// ========================================

client.on(
  Events.InteractionCreate,
  async (interaction) => {
    if (
      !interaction.isChatInputCommand() ||
      !interaction.guildId
    ) {
      return;
    }

    try {
      // ==================================
      // /ping
      // ==================================

      if (
        interaction.commandName ===
        "ping"
      ) {
        await interaction.reply(
          "🏓 Pong!",
        );

        return;
      }

      // ==================================
      // Owner-only commands
      // ==================================

      const ownerCommands = [
        "settings",
        "exempt-channel",
        "faq",
        "banword",
      ];

      if (
        ownerCommands.includes(
          interaction.commandName,
        )
      ) {
        const allowed =
          await requireOwner(
            interaction,
          );

        if (!allowed) {
          return;
        }
      }

      const settings =
        await getSettings(
          interaction.guildId,
        );

      // ==================================
      // /settings
      // ==================================

      if (
        interaction.commandName ===
        "settings"
      ) {
        const subcommand =
          interaction.options.getSubcommand();

        // ------------------------------
        // SHOW
        // ------------------------------

        if (
          subcommand === "show"
        ) {
          const antiLinks =
            settings.anti_link_enabled
              ? "🟢 مفعلة"
              : "🔴 متوقفة";

          const salam =
            settings.auto_salam_enabled
              ? "🟢 مفعل"
              : "🔴 متوقف";

          const faq =
            settings.faq_enabled
              ? "🟢 مفعل"
              : "🔴 متوقف";

          const ai =
            settings.ai_enabled
              ? "🟢 مفعل"
              : "🔴 متوقف";

          const hourly =
            settings.hourly_enabled
              ? "🟢 مفعل"
              : "🔴 متوقف";

          const general =
            settings.general_channel
              ? `<#${settings.general_channel}>`
              : "غير محددة";

          await interaction.reply(
            [
              "👑 **إعدادات 3RB Bot**",
              "",
              `🔗 حماية الروابط: ${antiLinks}`,
              `👋 رد السلام: ${salam}`,
              `❓ FAQ: ${faq}`,
              `🧠 AI: ${ai}`,
              `⏰ الرسائل التلقائية: ${hourly}`,
              `💬 الشات العام: ${general}`,
              "",
              `👑 المالك: <@${OWNER_ID}>`,
            ].join("\n"),
          );

          return;
        }

        // ------------------------------
        // ANTI LINKS
        // ------------------------------

        if (
          subcommand ===
          "anti-links"
        ) {
          const enabled =
            interaction.options.getBoolean(
              "enabled",
              true,
            );

          settings.anti_link_enabled =
            enabled ? 1 : 0;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            enabled
              ? "🛡️ تم تشغيل حماية الروابط."
              : "🛡️ تم إيقاف حماية الروابط.",
          );

          return;
        }

        // ------------------------------
        // AUTO SALAM
        // ------------------------------

        if (
          subcommand ===
          "auto-salam"
        ) {
          const enabled =
            interaction.options.getBoolean(
              "enabled",
              true,
            );

          settings.auto_salam_enabled =
            enabled ? 1 : 0;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            enabled
              ? "👋 تم تشغيل رد السلام."
              : "👋 تم إيقاف رد السلام.",
          );

          return;
        }

        // ------------------------------
        // FAQ
        // ------------------------------

        if (
          subcommand === "faq"
        ) {
          const enabled =
            interaction.options.getBoolean(
              "enabled",
              true,
            );

          settings.faq_enabled =
            enabled ? 1 : 0;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            enabled
              ? "❓ تم تشغيل FAQ."
              : "❓ تم إيقاف FAQ.",
          );

          return;
        }

        // ------------------------------
        // AI
        // ------------------------------

        if (
          subcommand === "ai"
        ) {
          const enabled =
            interaction.options.getBoolean(
              "enabled",
              true,
            );

          settings.ai_enabled =
            enabled ? 1 : 0;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            enabled
              ? "🧠 تم تشغيل الذكاء الاصطناعي."
              : "🧠 تم إيقاف الذكاء الاصطناعي.",
          );

          return;
        }

        // ------------------------------
        // HOURLY
        // ------------------------------

        if (
          subcommand ===
          "hourly"
        ) {
          const enabled =
            interaction.options.getBoolean(
              "enabled",
              true,
            );

          settings.hourly_enabled =
            enabled ? 1 : 0;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            enabled
              ? "⏰ تم تشغيل الرسائل التلقائية."
              : "⏰ تم إيقاف الرسائل التلقائية.",
          );

          return;
        }

        // ------------------------------
        // GENERAL
        // ------------------------------

        if (
          subcommand ===
          "general"
        ) {
          const channel =
            interaction.options.getChannel(
              "channel",
              true,
            );

          settings.general_channel =
            channel.id;

          await db
            .ref(
              `settings/${interaction.guildId}`,
            )
            .set(settings);

          await interaction.reply(
            `💬 تم تحديد <#${channel.id}> كقناة الشات العام.`,
          );

          return;
        }
      }

      // ==================================
      // /exempt-channel
      // ==================================

      if (
        interaction.commandName ===
        "exempt-channel"
      ) {
        const channel =
          interaction.options.getChannel(
            "channel",
            true,
          );

        const channels =
          exemptChannelIds(
            settings.exempt_channels,
          );

        if (
          !channels.includes(
            channel.id,
          )
        ) {
          channels.push(
            channel.id,
          );
        }

        settings.exempt_channels =
          channels.join(",");

        await db
          .ref(
            `settings/${interaction.guildId}`,
          )
          .set(settings);

        await interaction.reply(
          `🛡️ تم استثناء <#${channel.id}> من حماية الروابط.`,
        );

        return;
      }

      // ==================================
      // /faq
      // ==================================

      if (
        interaction.commandName ===
        "faq"
      ) {
        const subcommand =
          interaction.options.getSubcommand();

        const keyword =
          interaction.options
            .getString(
              "keyword",
              true,
            )
            .trim();

        const key =
          keyword.toLocaleLowerCase();

        if (
          subcommand === "add"
        ) {
          const answer =
            interaction.options
              .getString(
                "answer",
                true,
              )
              .trim();

          await db
            .ref(
              `faqs/${interaction.guildId}/${key}`,
            )
            .set({
              keyword,
              answer,
            });

          await interaction.reply(
            `✅ تم حفظ الرد للكلمة **${keyword}**.`,
          );

          return;
        }

        if (
          subcommand ===
          "delete"
        ) {
          await db
            .ref(
              `faqs/${interaction.guildId}/${key}`,
            )
            .remove();

          await interaction.reply(
            `🗑️ تم حذف الرد للكلمة **${keyword}**.`,
          );

          return;
        }
      }

      // ==================================
      // /banword
      // ==================================

      if (
        interaction.commandName ===
        "banword"
      ) {
        const subcommand =
          interaction.options.getSubcommand();

        const keyword =
          interaction.options
            .getString(
              "keyword",
              true,
            )
            .trim();

        const key =
          keyword.toLocaleLowerCase();

        if (
          subcommand === "add"
        ) {
          const duration =
            interaction.options.getInteger(
              "duration",
              true,
            );

          await db
            .ref(
              `banwords/${interaction.guildId}/${encodeURIComponent(
                key,
              )}`,
            )
            .set({
              keyword,
              duration,
            });

          await interaction.reply(
            `🔇 تم إضافة **${keyword}** إلى الكلمات الممنوعة.\n⏱️ مدة الكتم: **${duration} دقيقة**.`,
          );

          return;
        }

        if (
          subcommand ===
          "delete"
        ) {
          await db
            .ref(
              `banwords/${interaction.guildId}/${encodeURIComponent(
                key,
              )}`,
            )
            .remove();

          await interaction.reply(
            `✅ تم حذف **${keyword}** من الكلمات الممنوعة.`,
          );

          return;
        }
      }
    } catch (error) {
      console.error(
        "Interaction handling failed.",
        error,
      );

      const reply = {
        content:
          "❌ حدث خطأ أثناء تنفيذ الأمر.",

        flags:
          MessageFlags.Ephemeral,
      };

      if (
        interaction.replied ||
        interaction.deferred
      ) {
        await interaction
          .followUp(reply)
          .catch(() => {});
      } else {
        await interaction
          .reply(reply)
          .catch(() => {});
      }
    }
  },
);

// ========================================
// 💬 Messages
// ========================================

client.on(
  Events.MessageCreate,
  async (message) => {
    if (
      !message.guildId ||
      message.author.bot
    ) {
      return;
    }

    if (
      processedMessages.has(
        message.id,
      )
    ) {
      return;
    }

    processedMessages.add(
      message.id,
    );

    setTimeout(() => {
      processedMessages.delete(
        message.id,
      );
    }, 60000);

    try {
      // ==================================
      // Settings
      // ==================================

      const settings =
        await getSettings(
          message.guildId,
        );

      const isExempt =
        exemptChannelIds(
          settings.exempt_channels,
        ).includes(
          message.channelId,
        );

      // ==================================
      // 🔗 Anti Link
      // ==================================

      if (
        settings.anti_link_enabled &&
        !isExempt &&
        containsLinkOrInvite(
          message.content,
        ) &&
        !message.member?.permissions.has(
          PermissionFlagsBits.ManageMessages,
        )
      ) {
        await message
          .delete()
          .catch(() => {});

        await sendTemporaryWarning(
          message.channel,
        );

        return;
      }

      // ==================================
      // 🔇 Forbidden Words
      // ==================================

      if (
        message.content.trim() &&
        message.member &&
        !message.member.permissions.has(
          PermissionFlagsBits.ManageGuild,
        )
      ) {
        const banWords =
          await getBanWords(
            message.guildId,
          );

        const banWordsList =
          Object.values(
            banWords,
          );

        if (
          banWordsList.length >
          0
        ) {
          const normalizedContent =
            message.content
              .toLocaleLowerCase();

          const matchedWord =
            banWordsList.find(
              ({ keyword }) =>
                keyword &&
                normalizedContent.includes(
                  keyword.toLocaleLowerCase(),
                ),
            );

          if (matchedWord) {
            const duration =
              Number(
                matchedWord.duration,
              ) || 10;

            await message
              .delete()
              .catch(() => {});

            if (
              message.member
                .moderatable
            ) {
              const success =
                await message.member
                  .timeout(
                    duration *
                      60 *
                      1000,

                    `استخدام كلمة ممنوعة: ${matchedWord.keyword}`,
                  )
                  .then(
                    () => true,
                  )
                  .catch(
                    (error) => {
                      console.error(
                        "Failed to timeout member:",
                        error,
                      );

                      return false;
                    },
                  );

              if (success) {
                const warning =
                  await message.channel
                    .send(
                      `🔇 <@${message.author.id}> تم كتمك لمدة **${duration} دقيقة** بسبب استخدام كلمة ممنوعة.`,
                    )
                    .catch(
                      () => null,
                    );

                if (warning) {
                  setTimeout(
                    () => {
                      warning
                        .delete()
                        .catch(
                          () => {},
                        );
                    },
                    5000,
                  );
                }
              }
            }

            return;
          }
        }
      }

      // ==================================
      // ⭐ Level
      // ==================================

      if (
        isLevelQuestion(
          message.content,
        )
      ) {
        const userRef =
          db.ref(
            `levels/${message.guildId}/${message.author.id}`,
          );

        const userSnapshot =
          await userRef.once(
            "value",
          );

        if (
          !userSnapshot.exists()
        ) {
          await message.reply(
            `📊 <@${message.author.id}> مستواك حاليًا **1** ✨\nابدأ بالمشاركة في السيرفر عشان تجمع XP وترتقي! 🔥`,
          );
        } else {
          const userData =
            userSnapshot.val();

          const level =
            Number(
              userData.level,
            ) || 1;

          const xp =
            Number(
              userData.xp,
            ) || 0;

          const neededXp =
            level * 100;

          await message.reply(
            `📊 <@${message.author.id}> مستواك الحالي هو **${level}** ✨\n⭐ XP: **${xp} / ${neededXp}**`,
          );
        }

        return;
      }

      // ==================================
      // 🧠 AI
      // ==================================

      if (
        settings.ai_enabled &&
        message.mentions.has(
          client.user,
        )
      ) {
        const lastAiTime =
          aiCooldowns.get(
            message.author.id,
          ) ?? 0;

        if (
          Date.now() -
            lastAiTime <
          5000
        ) {
          return;
        }

        aiCooldowns.set(
          message.author.id,
          Date.now(),
        );

        await message.channel.sendTyping();

        const mentionRegex =
          new RegExp(
            `<@!?${client.user.id}>`,
            "g",
          );

        const prompt =
          message.content
            .replace(
              mentionRegex,
              "",
            )
            .trim();

        if (!prompt) {
          await message.reply(
            "هلا بك! أمُرني، كيف أقدر أساعدك اليوم في السيرفر؟ 🤍",
          );

          return;
        }

        try {
          const replyText =
            await askAI(prompt);

          if (
            replyText.length <=
            2000
          ) {
            await message.reply(
              replyText,
            );
          } else {
            const chunks =
              replyText.match(
                /[\s\S]{1,1900}/g,
              ) ?? [];

            for (
              let i = 0;
              i < chunks.length;
              i++
            ) {
              if (i === 0) {
                await message.reply(
                  chunks[i],
                );
              } else {
                await message.channel.send(
                  chunks[i],
                );
              }
            }
          }
        } catch (aiError) {
          console.error(
            "❌ All AI providers failed:",
            aiError,
          );

          const fallback =
            fallbackReplies[
              Math.floor(
                Math.random() *
                  fallbackReplies.length,
              )
            ];

          await message.reply(
            fallback,
          );
        }

        return;
      }

      // ==================================
      // 👋 Auto Salam
      // ==================================

      if (
        settings.auto_salam_enabled &&
        settings.general_channel ===
          message.channelId &&
        isSalam(
          message.content,
        )
      ) {
        const lastReply =
          salamCooldowns.get(
            message.guildId,
          ) ?? 0;

        if (
          Date.now() -
            lastReply >=
          30000
        ) {
          salamCooldowns.set(
            message.guildId,
            Date.now(),
          );

          await message.reply(
            "وعليكم السلام ورحمة الله وبركاته",
          );
        }
      }

      // ==================================
      // ❓ FAQ
      // ==================================

      if (
        settings.faq_enabled &&
        message.content.trim()
      ) {
        const faqsObj =
          await getFaqs(
            message.guildId,
          );

        const faqsList =
          Object.values(
            faqsObj,
          );

        if (
          faqsList.length > 0
        ) {
          faqsList.sort(
            (
              first,
              second,
            ) =>
              second.keyword
                .length -
              first.keyword.length,
          );

          const normalizedContent =
            message.content
              .toLocaleLowerCase();

          const faq =
            faqsList.find(
              ({ keyword }) =>
                normalizedContent.includes(
                  keyword.toLocaleLowerCase(),
                ),
            );

          if (faq) {
            await message.reply(
              faq.answer,
            );
          }
        }
      }

      // ==================================
      // ⭐ XP
      // ==================================

      const cooldownKey =
        `${message.guildId}_${message.author.id}`;

      const lastXpTime =
        xpCooldowns.get(
          cooldownKey,
        ) ?? 0;

      if (
        Date.now() -
          lastXpTime >
        60000
      ) {
        xpCooldowns.set(
          cooldownKey,
          Date.now(),
        );

        const userRef =
          db.ref(
            `levels/${message.guildId}/${message.author.id}`,
          );

        const userSnapshot =
          await userRef.once(
            "value",
          );

        let userData =
          userSnapshot.exists()
            ? userSnapshot.val()
            : {
                xp: 0,
                level: 1,
              };

        userData.xp +=
          Math.floor(
            Math.random() * 10,
          ) + 15;

        const neededXp =
          userData.level *
          100;

        if (
          userData.xp >=
          neededXp
        ) {
          userData.level += 1;

          userData.xp = 0;

          if (
            levelUpChannelId
          ) {
            const levelChannel =
              await message.guild.channels
                .fetch(
                  levelUpChannelId,
                )
                .catch(
                  () => null,
                );

            if (
              levelChannel?.isTextBased()
            ) {
              await levelChannel.send(
                `💎 **| ترقية جديدة!**\n` +
                  `ارتقى العضو <@${message.author.id}> للمستوى **${userData.level}** ✨\n\n` +
                  `⚔️~~~~~~~~~~~~~~~~~~~~~~~~~~~~⚔️\n` +
                  `تشتعل الهمم وتصعد الأسماء نحو القمة في سماء المملكة.\n` +
                  `أطلق من يتواجد عندنا.. وحياك الله ! 🦅🔥`,
              );
            }
          }
        }

        await userRef.set(
          userData,
        );
      }
    } catch (error) {
      console.error(
        "Message handling failed.",
        error,
      );
    }
  },
);

// ========================================
// 🛑 Graceful Shutdown
// ========================================

async function shutdown(
  signal,
) {
  console.info(
    `Received ${signal}; shutting down.`,
  );

  client.destroy();

  await new Promise(
    (resolve) =>
      webServer.close(resolve),
  );

  process.exit(0);
}

process.once(
  "SIGINT",
  () =>
    void shutdown(
      "SIGINT",
    ),
);

process.once(
  "SIGTERM",
  () =>
    void shutdown(
      "SIGTERM",
    ),
);

// ========================================
// 🚀 Start
// ========================================

try {
  await registerCommands();

  await client.login(token);

  console.info(
    "✅ Discord bot successfully logged in.",
  );
} catch (error) {
  console.error(
    "❌ Discord bot failed to start.",
    error,
  );

  webServer.close();

  process.exitCode = 1;
}
