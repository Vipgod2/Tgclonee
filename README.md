import asyncio
import re
from telethon import TelegramClient, events

# === CONFIG ===
api_id = 23416992
api_hash = 'bebe7b8b7bc74aeaa08f658e8aa799fe'
bot_tokens = [
    '8996998551:AAGkXd6WO-Kf7WHOkIjA4ZC_c8R14upraDQ',
    #'8361725713:AAFAvAzrhTDpLib4pK__C36EGZvemBHbMyM',
    #'8350221175:AAFxpkg9BASmXJpd-xdIQID7451e-TW4rb8',
    # Add more tokens here
]
# ===============

bot_clients = []


def split_range(start, end, parts):
    step = (end - start + 1) // parts
    ranges = []
    for i in range(parts):
        s = start + i * step
        e = (start + (i + 1) * step - 1) if i < parts - 1 else end
        ranges.append((s, e))
    return ranges


async def forward_range(client, bot_index, chat, start_id, end_id, target_chat_id):
    print(f"🚀 Bot {bot_index} handling range {start_id} to {end_id}")
    
    try:
        src_entity = await client.get_entity(chat)
    except Exception as e:
        print(f"❌ Bot {bot_index} cannot access chat {chat}: {e}")
        return

    for msg_id in range(start_id, end_id + 1):
        try:
            msg = await client.get_messages(src_entity, ids=msg_id)
            if msg and msg.media:
                await client.send_file(target_chat_id, file=msg.media)
            elif msg and msg.text:
                await client.send_message(target_chat_id, msg.text)
            await asyncio.sleep(1.2)
        except Exception as e:
            print(f"❌ Bot {bot_index} failed on msg {msg_id}: {e}")
            continue
    print(f"✅ Bot {bot_index} finished range {start_id}-{end_id}")


async def main():
    # Start all bot sessions
    for i, token in enumerate(bot_tokens):
        session_name = f'bot_session_{i + 1}'
        client = TelegramClient(session_name, api_id, api_hash)
        await client.start(bot_token=token)
        bot_clients.append((i + 1, client))

    print(f"✅ {len(bot_clients)} bots are ready.")

    command_bot_index, command_client = bot_clients[0]

    @command_client.on(events.NewMessage(pattern=r'/kidilo ([\S]+)\s+(\d+)\s+(\d+)'))
    async def handle_command(event):
        chat_input = event.pattern_match.group(1)
        start_id = int(event.pattern_match.group(2))
        end_id = int(event.pattern_match.group(3))
        target_chat_id = event.chat_id

        if chat_input.startswith("https://t.me/"):
            match = re.match(r'https?://t\.me/([^/]+)/?', chat_input)
            if not match:
                return await event.reply("❌ Invalid link.")
            chat = match.group(1)
        else:
            chat = int(chat_input) if chat_input.lstrip("-").isdigit() else chat_input

        chunks = split_range(start_id, end_id, len(bot_clients))

        tasks = []
        for i, (bot_index, client) in enumerate(bot_clients):
            s_id, e_id = chunks[i]
            tasks.append(forward_range(client, bot_index, chat, s_id, e_id, target_chat_id))

        await event.reply(f"🚀 Forwarding {start_id} to {end_id} using {len(bot_clients)} bots...")
        await asyncio.gather(*tasks)
        await event.reply("✅ All bots finished forwarding.")

    await command_client.run_until_disconnected()


asyncio.run(main())
