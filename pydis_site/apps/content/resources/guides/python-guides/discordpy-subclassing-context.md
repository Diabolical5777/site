---
title: Subclassing Context in discord.py
description: "Subclassing the default `commands.Context` class to add more functionability and customizability."
---

Start by reading the guide on [subclassing the `Bot` class](./subclassing_bot.md). A subclass of Bot has to be used to
inject your custom context subclass into discord.py.

## Overview

The way this works is by creating a subclass of discord.py's [`Context` class](https://discordpy.readthedocs.io/en/latest/ext/commands/api.html#discord.ext.commands.Context)
adding whatever functionality you wish. Usually this is done by adding custom methods or properties, so that you don't need to
copy it around or awkwardly import it elsewhere.

This guide will show you how to add a `prompt()` method to the context and how to use it in a command.

## Example subclass and code

The first part - of course - is creating the actual context subclass. This is done similarly to creating a bot
subclass, it will look like this:

```python
import asyncio
from typing import Optional

from discord import RawReactionActionEvent
from discord.ext import commands


class CustomContext(commands.Context):
    async def prompt(
            self,
            message: str,
            *,
            timeout=30.0,
            delete_after=True
    ) -> Optional[bool]:
        """Prompt the author with an interactive confirmation message.

        This method will send the `message` content, and wait for max `timeout` seconds
        (default is `30`) for the author to react to the message.

        If `delete_after` is `True`, the message will be deleted before returning a
        True, False, or None indicating whether the author confirmed, denied,
        or didn't interact with the message.
        """
        msg = await self.send(message)

        for reaction in ('✅', '❌'):
            await msg.add_reaction(reaction)

        confirmation = None

        # This function is a closure because it is defined inside of another
        # function. This allows the function to access the self and msg
        # variables defined above.

        def check(payload: RawReactionActionEvent):
            # 'nonlocal' works almost like 'global' except for functions inside of
            # functions. This means that when 'confirmation' is changed, that will
            # apply to the variable above
            nonlocal confirmation

            if payload.message_id != msg.id or payload.user_id != self.author.id:
                return False

            emoji = str(payload.emoji)

            if emoji == '✅':
                confirmation = True
                return True

            elif emoji == '❌':
                confirmation = False
                return True

            # This means that it was neither of the two emojis added, so the author
            # added some other unrelated reaction.
            return False

        try:
            await self.bot.wait_for('raw_reaction_add', check=check, timeout=timeout)
        except asyncio.TimeoutError:
            # The 'confirmation' variable is still None in this case
            pass

        if delete_after:
            await msg.delete()

        return confirmation
```

After creating your context subclass, you need to override the `get_context()` method on your
Bot class and change the default of the `cls` parameter to this subclass:

```python
from discord.ext import commands


class CustomBot(commands.Bot):
    async def get_context(self, message, *, cls=CustomContext):  # From the above codeblock
        return await super().get_context(message, cls=cls)
```

Now that discord.py is using your custom context, you can use it in a command. For example:

```python
import discord
from discord.ext import commands

# Enable the message intent so that we get message content. This is needed for
# the commands we define below
intents = discord.Intents.default()
intents.message_content = True


# Replace '...' with any additional arguments for the bot
bot = CustomBot(intents=intents, ...)


@bot.command()
async def massban(ctx: CustomContext, members: commands.Greedy[discord.Member]):
    prompt = await ctx.prompt(f"Are you sure you want to ban {len(members)} members?")
    if not prompt:
        # Return if the author cancelled, or didn't react in time
        return

    ...  # Perform the mass-ban, knowing the author has confirmed this action
```
