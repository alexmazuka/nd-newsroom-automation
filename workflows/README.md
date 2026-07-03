# ⚙️ Воркфлоу n8n: запуск і тестування

Три готові воркфлоу покривають ядро конвеєра (М1–М4). Імпорт і перший тест — близько 15 хвилин.

## Крок 0. Розгорнути n8n

**Варіант А — свій VPS (рекомендовано, ~$5–6/міс):**
```bash
docker volume create n8n_data
docker run -d --restart unless-stopped --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Kyiv" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
**Варіант Б — без VPS:** n8n Cloud (тріал) для тестів, або Make.com free (1000 операцій/міс) як фолбек — логіка воркфлоу переноситься 1:1.

## Крок 1. Зібрати доступи

| Що | Де взяти | Куди вставити |
|---|---|---|
| Telegram-бот (token) | @BotFather → `/newbot` | n8n → Credentials → Telegram API |
| Chat ID чату кураторів | додати бота в чат, id — через @userinfobot | вузол Telegram: `ВСТАВТЕ_CHAT_ID` |
| Anthropic API key | console.anthropic.com → API Keys (одразу поставте ліміт витрат) | вузол Claude: `ВСТАВТЕ_ANTHROPIC_API_KEY` |
| WP Application Password | WP-адмінка → Користувачі → Профіль → Application Passwords | n8n → Credentials → Basic Auth (логін WP + пароль) |

⚠️ **Після першого тесту** перенесіть API-ключ Anthropic із поля заголовка у n8n Credential (тип *Header Auth*, header `x-api-key`) — ключі не повинні лежати у тілі воркфлоу.

## Крок 2. Імпортувати воркфлоу

n8n → **Workflows → ⋯ → Import from File** → оберіть JSON:

| Файл | Модулі | Що робить | Тригер |
|---|---|---|---|
| `01-news-intake.json` | М1+М2 | RSS → антидубль (журнал 72 год) → картка новини в Telegram кураторів | розклад, кожні 15 хв |
| `02-draft-to-wordpress.json` | М3 | факти → Claude (драфт + 3 заголовки + SEO-фраза) → **чернетка** у WordPress | webhook `nd-draft` |
| `03-publish-to-social.json` | М4 | публікація на сайті → Claude (версії FB/TG/X) → куратору SMM у Telegram | webhook `nd-social` |

## Крок 3. Тестові запуски

**01 — збір новин:** відкрийте воркфлоу → Execute workflow. У чаті кураторів з'являться картки свіжих матеріалів з RSS. Запустіть ще раз — карток не буде (антидубль працює).

**02 — AI-чернетка:** натисніть Execute workflow (вебхук слухає), потім:
```bash
curl -X POST "<Test URL вузла Webhook>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Обстріл Костянтинівки",
    "facts": "- 3 липня близько 14:00 обстріляно центр міста (за даними ОВА)\n- пошкоджено пятиповерхівку по вул. Центральній\n- інформація про постраждалих уточнюється",
    "source": "Донецька ОВА"
  }'
```
Перевірте: у WP-адмінці → Записи → Чернетки лежить драфт із заголовками, SEO-фразою і позначками `[ПЕРЕВІРИТИ]` там, де фактів бракує.

**03 — соцадаптація:** аналогічно, curl із `post_title` / `excerpt` / `post_permalink` (приклад у стікері воркфлоу). Для бойового режиму поставте на WordPress плагін **WP Webhooks** і налаштуйте тригер «Post published» на Production URL вебхука.

## Крок 4. Поля Yoast через REST (для повної автоматизації М3)

Щоб воркфлоу 02 міг заповнювати SEO-поля Yoast, додайте у `functions.php` дочірньої теми (або через WPVibe):

```php
add_action('init', function () {
    foreach (['_yoast_wpseo_title', '_yoast_wpseo_metadesc', '_yoast_wpseo_focuskw'] as $key) {
        register_post_meta('post', $key, [
            'show_in_rest'  => true,
            'single'        => true,
            'type'          => 'string',
            'auth_callback' => function () { return current_user_can('edit_posts'); },
        ]);
    }
});
```

Після цього у `jsonBody` вузла WordPress можна передавати `meta: { _yoast_wpseo_focuskw: '...', _yoast_wpseo_metadesc: '...' }`.

## Вибір моделі Claude

У воркфлоу стоїть `claude-opus-4-8` (максимальна якість). Для економії змініть поле `model`:
- `claude-sonnet-5` — баланс (інтро-ціни до 31.08.2026)
- `claude-haiku-4-5` — найдешевша, достатня для соцадаптації і заголовків

Повна таблиця цін — у [головному README](../README.md#моделі-claude-і-вартість).
