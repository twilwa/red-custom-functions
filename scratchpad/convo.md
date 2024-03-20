class Conversation(AssistantBaseModel):
    messages: List[dict] = []
    last_updated: float = 0.0
    system_prompt_override: Optional[str] = None

    def function_count(self) -> int:
        if not self.messages:
            return 0
        return sum(i["role"] in ["function", "tool"] for i in self.messages)

    def is_expired(self, conf: GuildSettings, member: Optional[discord.Member] = None):
        if not conf.get_user_max_time(member):
            return False
        return (datetime.now().timestamp() - self.last_updated) > conf.get_user_max_time(member)

    def cleanup(self, conf: GuildSettings, member: Optional[discord.Member] = None):
        clear = [
            self.is_expired(conf, member),
            not conf.get_user_max_retention(member),
        ]
        if any(clear):
            self.messages.clear()
        elif conf.max_retention:
            self.messages = self.messages[-conf.get_user_max_retention(member) :]

    def reset(self):
        self.refresh()
        self.messages.clear()

    def refresh(self):
        self.last_updated = datetime.now().timestamp()

    def overwrite(self, messages: List[dict]):
        self.refresh()
        self.messages = [i for i in messages if i["role"] != "system"]

    def update_messages(
        self,
        message: str,
        role: str,
        name: str = None,
        tool_id: str = None,
        position: int = None,
    ) -> None:
        """Update conversation cache

        Args:
            message (str): the message
            role (str): 'system', 'user' or 'assistant'
            name (str): the name of the bot or user
            position (int): the index to place the message in
        """
        message: dict = {"role": role, "content": message}
        if name:
            message["name"] = name
        if tool_id:
            message["tool_call_id"] = tool_id
        if position:
            self.messages.insert(position, message)
        else:
            self.messages.append(message)
        self.refresh()

    def prepare_chat(
        self,
        user_message: str,
        initial_prompt: str,
        system_prompt: str,
        name: str = None,
        images: List[str] = None,
        resolution: str = "low",
    ) -> List[dict]:
        """Pre-appends the prmompts before the user's messages without motifying them"""
        prepared = []
        if system_prompt.strip():
            prepared.append({"role": "system", "content": system_prompt})
        if initial_prompt.strip():
            prepared.append({"role": "user", "content": initial_prompt})
        prepared.extend(self.messages)

        if images:
            content = [{"type": "text", "text": user_message}]
            for img in images:
                if img.lower().startswith("http"):
                    content.append({"type": "image_url", "image_url": img})
                else:
                    content.append(
                        {
                            "type": "image_url",
                            "image_url": {"url": f"data:image/jpeg;base64,{img}", "detail": resolution},
                        }
                    )

        else:
            content = user_message

        user_message_payload = {"role": "user", "content": content}
        if name:
            user_message_payload["name"] = name
        prepared.append(user_message_payload)
        self.messages.append(user_message_payload)
        self.refresh()
        return prepared