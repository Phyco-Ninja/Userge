import re
import string

from pyrogram.types import ChatPermissions, ChatMemberUpdated

from userge import userge, filters, Message, get_collection


PUNCTUATIONS: re.Pattern = re.compile(
    "|".join(
        [re.escape(x) for x in string.punctuation]
    )
)
MODES: dict = {
    1: ("User Kicked", userge.kick_chat_member, {"until_date": 60}),
    2: ("User Banned", userge.kick_chat_member, {"until_date": 0}),
    3: ("User Muted", userge.restrict_chat_member, {"permissions": ChatPermissions()})
}


class BlackList:

    def __init__(self, client, collection):
        self.collection = collection
        self.client = client
        self.chats = filters.chat([])
        self.logger = self.client.getCLogger("BlackList")
        self.alerts = {}

    async def add_word(self, chat_id: int, word: str):
        """ blacklist a word in specific chat """
        word = word.lower().strip()
        words = []
        data = await self.collection.find_one({"chat_id": chat_id})
        if data:
            words = data.get("words", [])
        if word in words:
            return True
        words.append(word)
        await self.logger.log(f"{word} added in `{chat_id}`")
        return await self.collection.update_one(
            {"chat_id": chat_id}, {"$set": {"words": words}}, upsert=True
        )

    async def delete_word(self, chat_id: int, word: str):
        """ remove a blacklist """
        word = word.lower().strip()
        data = await self.collection.find_one({"chat_id": chat_id})
        if data:
            words = data.get("words", [])
            words.remove(word)
            await self.logger.log(f"{word} removed in `{chat_id}`")
            return await self.collection.update_one(
                {"chat_id": chat_id}, {"$set": {"words": words}}
            )

    async def load(self):
        """ initialize chat filter """
        async for data in self.collection.find({}):
            chat = data.get("chat_id")
            if chat:
                self.chats.add(chat)

    async def get_words(self, chat_id: int):
        """ all blacklisted words in chat """
        data = await self.collection.find_one({"chat_id": chat_id})
        if data:
            return data.get("words", [])
        return []

    async def search(self, message: Message):
        """ search for blacklist in update """
        content = message.text or message.caption
        if not content:
            return False, None
        for bld in await self.get_words(message.chat.id):
            pattern = re.compile(f"(?i)([^a-zA-Z]|^)({re.escape(bld)})([^a-zA-Z]|$)")
            if pattern.search(content):
                return True, bld
            if pattern.search(PUNCTUATIONS.sub("", content)):
                return True, bld
        return False, None

    async def set_mode(self, chat_id: int, mode: int):
        return await self.collection.update_one(
            {"_id": f"SETTINGS_{chat_id}"}, {"$set": {"action": int(mode)}}, upsert=True
        )

    async def get_mode(self, chat_id: int):
        data = await self.collection.find_one({"_id": f"SETTINGS_{chat_id}"})
        if data:
            return MODES.get(data.get("action", 0))
        return None

    async def set_chat_alerts(self, chat_id: int, mode: int):
        self.alerts[chat_id] = mode
        return await self.collection.update_one(
            {"_id": f"SETTINGS_{chat_id}"}, {"$set": {"alerts": mode}}, upsert=True
        )

    async def get_chat_alerts(self, chat_id: int):
        mode = self.alerts.get(chat_id)
        if mode is None:
            data = await self.collection.find_one({'_id': f"SETTINGS_{chat_id}"})
            mode = data.get("alerts", 1)
        return bool(mode)

    async def action(self, chat_id: int, user_id: int):
        mode = await self.get_mode(chat_id)
        if mode is None:
            return ""
        await mode[1](chat_id, user_id, **mode[2])
        return mode[0]


Blacklister = BlackList(userge, get_collection("BLACKLIST"))

# Create an admin cache
# contains ChatMember object of admins of chat including client
CHAT_ADMINS = {}


async def load_admin(chat_id):
    CHAT_ADMINS[chat_id] = [
        x async for x in userge.iter_chat_members(
            chat_id, filter="administrators"
        )
    ]
    return CHAT_ADMINS[chat_id]


async def _init():
    await Blacklister.load()
    # Load admins on startup
    for chat_id in Blacklister.chats:
        await load_admin(chat_id)



def admin_type(status: str):
    return status in ("administrator", "creator")


async def get_chat_admin(chat_id: int, user_id: int):
    admins = CHAT_ADMINS.get(chat_id)
    if admins is None:
        admins = await load_admin(chat_id)
        
    for admin in admins:
        if admin.user.id == user_id:
            return admin
    return None


# Actively update admin cache on every new chat member update
@userge.on_chat_member_updated(group=-4)
async def _chat_admin_updater(_client: userge, update: ChatMemberUpdated):
    userge.getLogger("CMU").info("Chat Member Update Received")
    _old = update.old_chat_member
    new = update.new_chat_member
    if new is None:
        return
    user_id = new.user.id
    chat_id = update.chat.id
    admin = await get_chat_admin(chat_id, user_id)

    # new admin.
    if admin is None and admin_type(new.status):
        CHAT_ADMINS.get(chat_id, []).append(new)
        return

    if admin:
        # demoted / kicked admin
        if not admin_type(new.status):
            CHAT_ADMINS[chat_id].remove(admin)
        # new perms
        else:
            CHAT_ADMINS[chat_id].remove(admin)
            CHAT_ADMINS[chat_id].append(new)
    update.continue_propagation()


@userge.on_cmd("blerts", about={
    'header': "Blacklist Alerts",
    'description': "set or get blacklist status",
    'flags': {"-y": "enable", "-n": "disable"},
    'usage': "{tr}blerts [opt flags]"
})
async def blserts(message: Message):
    if '-y' in message.flags:
        out = "Blacklist Alerts Enabled"
        await Blacklister.set_chat_alerts(message.chat.id, 1)
    elif '-n' in message.flags:
        out = "Blacklist Alerts Disabled"
        await Blacklister.set_chat_alerts(message.chat.id, 0)
    else:
        out = f"Blacklist Alert: `{await Blacklister.get_chat_alerts(message.chat.id)}`"
    await message.edit(out)


@userge.on_cmd("blmode", about={
    'header': "Blacklist Mode",
    'description': "Set blacklist mode to ban/kick/mute user",
    'options': {
        1: 'Kick User',
        2: 'Ban User',
        3: 'Mute User',
        0: 'Delete Messages only'
    }
})
async def set_blacklist_mode(message: Message):
    mode = message.input_str
    if not mode.isdigit():
        return await message.err("Only integers are supported")
    await Blacklister.set_mode(message.chat.id, int(mode))
    await message.edit("Blacklist mode updated")


@userge.on_cmd("cblmode", about={
    'header': "View Mode",
    'description': "View Current blacklist Mode"
})
async def veiw_mode_(message: Message):
    mode = await Blacklister.get_mode(message.chat.id)
    if mode is None:
        return await message.edit("Blacklist Mode: `Deleting Messages`")
    if 'Kick' in mode[0]:
        out = 'Blacklist Mode: `Kicking Users`'
    elif 'Ban' in mode[0]:
        out = 'Blacklist Mode: `Banning Users`'
    else:
        out = 'Blacklist Mode: `Muting Users`'
    await message.edit(out)


@userge.on_cmd("blacklists", about={
    'header': "List Blacklist",
    'description': "List all blacklisted words in a chat"
})
async def list_blads(message: Message):
    out = ""
    for word in await Blacklister.get_words(message.chat.id):
        out += f"  ~ `{word}`\n"
    if out == "":
        return await message.err("Empty like your brain.")
    await message.edit(
        f"**BlackLists in this chat**:\n\n{out}"
    )


@userge.on_cmd("blacklist", about={
    'header': "Add Blacklist",
    'description': "Blacklist a word in specific chat",
})
async def add_bls(message: Message):
    inp = message.input_str
    if not inp:
        return await message.err("@_@ yea? what?")
    await Blacklister.add_word(message.chat.id, inp)
    await Blacklister.load()
    await message.edit("Okay. Blacklisted")


@userge.on_cmd("rmblack", about={
    'header': "Remove blacklist",
    'description': "Remove a blacklist"
})
async def go_away_blacklist(message: Message):
    inp = message.input_str
    if not inp:
        return await message.err("@_@ yea? what?")
    await Blacklister.delete_word(message.chat.id, inp)
    await message.edit("Okay. Blacklist removed")


STUPIX: dict = {}


@userge.on_filters(Blacklister.chats & ~filters.me)
async def handle_update_black(message: Message):
    if message.sender_chat:
        return
    if message.from_user is None:
        return
    if await get_chat_admin(message.chat.id, message.from_user.id):
        return
    success, word = await Blacklister.search(message)
    if not success:
        return
    # pylint: disable=W0212
    await message.forward(chat_id=Blacklister.logger._id)
    deleh = await message.delete()
    try:
        out = await Blacklister.action(message.chat.id, message.from_user.id)
        if out:
            out = f"\nAction: {out}"
    except Exception as e:
        await Blacklister.logger.log(f"Unhandled Error {type(e).__name__}: `{e}`")
        out = ""
    if not deleh and out == '':
        return
    last_alert = STUPIX.get(message.chat.id)
    if (
        last_alert and
        last_alert[0] == message.from_user.id and
        last_alert[1] == word
    ):
        return
    # if alerts enabled sent a message
    if await Blacklister.get_chat_alerts(message.chat.id):
        await message.edit(
            f"{message.from_user.mention} has successfully triggered a"
            " blacklisted word. Congratulations your message "
            f"was deleted. {out}"
        )
    # log activity in Channel
    await Blacklister.logger.log(
        f"{message.from_user.mention} triggered `{word}`{out}"
    )
    STUPIX[message.chat.id] = (message.from_user.id, word)
