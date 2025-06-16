# Installation of Conversation Starter for HomeAssistantAssist

> ⚠️ **Before continuing**, ensure everything is already working with the original Assist integration.

Original installation instructions here: https://github.com/fabianosan/HomeAssistantAssist/blob/main/doc/en/INSTALLATION.md

---

## 1. Update Lambda Function

Replace the current Lambda function with the updated version provided in the `code` section of this repository (within the Alexa skill).

```
# -*- coding: utf-8 -*-
import os
import re
import logging
import json
import random
import uuid
import requests
import requests.exceptions
import ask_sdk_core.utils as ask_utils

from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler, AbstractExceptionHandler
from ask_sdk_model.interfaces.alexa.presentation.apl import RenderDocumentDirective, ExecuteCommandsDirective, OpenUrlCommand
from datetime import datetime, timezone, timedelta

# Load configurations and localization
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

def load_config(file_name):
    if str(file_name).endswith(".lang") and not os.path.exists(file_name):
        file_name = "locale/en-US.lang"
    try:
        with open(file_name, encoding='utf-8') as f:
            for line in f:
                line = line.strip()
                if not line or '=' not in line:
                    continue
                name, value = line.split('=', 1)
                globals()[name] = value
    except Exception as e:
        logger.error(f"Error loading file: {str(e)}")

# Initial config load
load_config("config.cfg")
load_config("locale/en-US.lang")

# Validate HA settings
if not globals().get("home_assistant_url") or not globals().get("home_assistant_token"):
    raise ValueError("home_assistant_url or home_assistant_token configuration are not set!")

# Globals for conversation
conversation_id = None
last_interaction_date = None
is_apl_supported = False
apl_document_token = str(uuid.uuid4())

# Helper: fetch text input via webhook
def fetch_prompt_from_ha():
    """
    Reads the state of your input_text helper directly via REST.
    """
    try:
        entity = globals().get("assist_input_entity", "input_text.assistant_input")
        url = f"{globals().get('home_assistant_url')}/api/states/{entity}"
        headers = {
            "Authorization": f"Bearer {globals().get('home_assistant_token')}",
            "Content-Type": "application/json"
        }
        resp = requests.get(url, headers=headers, timeout=5)
        if resp.status_code == 200:
            return resp.json().get("state", "").strip()
        else:
            logger.error(f"HA state fetch failed: {resp.status_code} {resp.text}")
    except Exception as e:
        logger.error(f"Error fetching prompt from HA state: {e}")
    return ""


class LaunchRequestHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return ask_utils.is_request_type("LaunchRequest")(handler_input)

    def handle(self, handler_input):
        global conversation_id, last_interaction_date, is_apl_supported
        # Load locale per user
        locale = handler_input.request_envelope.request.locale
        load_config(f"locale/{locale}.lang")

                # Check for a pre-set prompt from HA
        prompt = fetch_prompt_from_ha()
        # Only treat valid prompts that are not the literal "none"
        if prompt and prompt.lower() != "none":
            # Process this prompt as user input and keep session open for follow-up
            response = process_conversation(prompt)
            return handler_input.response_builder\
                .speak(response)\
                .ask(globals().get("alexa_speak_question"))\
                .response

        # No prompt: proceed with default welcome
        device = handler_input.request_envelope.context.system.device
        is_apl_supported = device.supported_interfaces.alexa_presentation_apl is not None
        if is_apl_supported:
            handler_input.response_builder.add_directive(
                RenderDocumentDirective(token=apl_document_token, document=load_template("apl_openha.json"))
            )
        now = datetime.now(timezone(timedelta(hours=-3)))
        current_date = now.strftime('%Y-%m-%d')
        speak_output = globals().get("alexa_speak_next_message")
        if last_interaction_date != current_date:
            speak_output = globals().get("alexa_speak_welcome_message")
            last_interaction_date = current_date
        return handler_input.response_builder.speak(speak_output).ask(speak_output).response

class GptQueryIntentHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return ask_utils.is_intent_name("GptQueryIntent")(handler_input)

    def handle(self, handler_input):
        query = handler_input.request_envelope.request.intent.slots["query"].value
        logger.info(f"Query received from Alexa: {query}")
        response = keywords_exec(query, handler_input)
        if response:
            return response
        device_id = ""
        if globals().get("home_assistant_room_recognition", "").lower() == "true":
            device_id = ". device_id: " + handler_input.request_envelope.context.system.device.device_id
        response = process_conversation(f"{query}{device_id}")
        logger.info(f"Response generated: {response}")
        return handler_input.response_builder.speak(response).ask(globals().get("alexa_speak_question")).response

# Trata palavras chaves para executar comandos específicos
def keywords_exec(query, handler_input):
    # Se o usuário der um comando para 'abrir dashboard' ou 'abrir home assistant', abre o dashboard e interrompe a skill
    keywords_top_open_dash = globals().get("keywords_to_open_dashboard").split(";")
    if any(ko.strip().lower() in query.lower() for ko in keywords_top_open_dash):
        logger.info("Opening Home Assistant dashboard")
        open_page(handler_input)
        return handler_input.response_builder.speak(globals().get("alexa_speak_open_dashboard")).response
    
    # Se o usuário der um comando de agradecimento o upara sair, interrompe a skill
    keywords_close_skill = globals().get("keywords_to_close_skill").split(";")
    if any(kc.strip().lower() in query.lower() for kc in keywords_close_skill):
        logger.info("Closing skill from keyword command")
        return CancelOrStopIntentHandler().handle(handler_input)
    
    # Se não for uma palavra-chave, continua o fluxo normalmente
    return None

# Chama a API do Home Assistant e trata a resposta
def process_conversation(query):
    global conversation_id
    
    home_assistant_agent_id = globals().get("home_assistant_agent_id", None)
    home_assistant_language = globals().get("home_assistant_language", None)
        
    try:
        headers = {
            "Authorization": "Bearer {}".format(globals().get("home_assistant_token")),
            "Content-Type": "application/json",
        }
        data = {
            "text": replace_words(query)
        }
        # Adding optional parameters to request
        if home_assistant_language:
            data["language"] = home_assistant_language
        if home_assistant_agent_id:
            data["agent_id"] = home_assistant_agent_id
        if conversation_id:
            data["conversation_id"] = conversation_id

        # Faz a requisição na API do Home Assistant
        ha_api_url = "{}/api/conversation/process".format(globals().get("home_assistant_url"))
        logger.debug(f"HA request url: {ha_api_url}")        
        logger.debug(f"HA request data: {data}")
        
        response = requests.post(ha_api_url, headers=headers, json=data, timeout=7)
        
        logger.debug(f"HA response status: {response.status_code}")
        logger.debug(f"HA response data: {response.text}")
        
        contenttype = response.headers.get('Content-Type', '')
        logger.debug(f"Content-Type: {contenttype}")
        
        if (contenttype == "application/json"):
            response_data = response.json()
            if response.status_code == 200 and "response" in response_data:
                conversation_id = response_data.get("conversation_id", conversation_id)
                response_type = response_data["response"]["response_type"]
                
                if response_type == "action_done" or response_type == "query_answer":
                    speech = response_data["response"]["speech"]["plain"]["speech"]
                    # Remover "device_id:" e o que vem depois
                    if "device_id:" in speech:
                        speech = speech.split("device_id:")[0].strip()
                elif response_type == "error":
                    speech = response_data["response"]["speech"]["plain"]["speech"]
                    logger.error(f"Error code: {response_data['response']['data']['code']}")
                else:
                    speech = globals().get("alexa_speak_error")
                
            return improve_response(speech)
        elif (contenttype == "text/html") and int(response.status_code, 0) >= 400:
            errorMatch = re.search(r'<title>(.*?)</title>', response.text, re.IGNORECASE)
            
            if errorMatch:
                title = errorMatch.group(1)
                logger.error(f"HTTP error {response.status_code} ({title}): Unable to connect to your Home Assistant server")
            else:
                logger.error(f"HTTP error {response.status_code}: Unable to connect to your Home Assistant server. \n {response.text}")
                
            return globals().get("alexa_speak_error")
        elif  (contenttype == "text/plain") and int(response.status_code, 0) >= 400:
            logger.error(f"Error processing request: {response.text}")
            return globals().get("alexa_speak_error")
        else:
            logger.error(f"Error processing request: {response.text}")
            return globals().get("alexa_speak_error")
            
    except requests.exceptions.Timeout as te:
        # Tratamento para timeout
        logger.error(f"Timeout when communicating with Home Assistant: {str(te)}", exc_info=True)
        return globals().get("alexa_speak_timeout")

    except Exception as e:
        logger.error(f"Error processing response: {str(e)}", exc_info=True)
        return globals().get("alexa_speak_error")

# Substitui palavras geradas incorretamente pelo interpretador da Alexa na query
def replace_words(query):
    query = query.replace('4.º','quarto')
    return query

# Substitui palavras e caracteres especiais para falar a resposta da API
def improve_response(speech):
    # Função para melhorar a legibilidade da resposta
    speech = speech.replace(':\n\n', '').replace('\n\n', '. ').replace('\n', ',').replace('-', '').replace('_', ' ')
    
    #replacements = str.maketrans('ïöüÏÖÜ', 'iouIOU')
    #speech = speech.translate(replacements)
    
    speech = re.sub(r'[^A-Za-z0-9çÇáàâãäéèêíïóôõöúüñÁÀÂÃÄÉÈÊÍÏÓÔÕÖÚÜÑ\sß.,!?]', '', speech)
    return speech

# Carrega o template do APL da tela inicial
def load_template(filepath):
    # Carrega o template da interface gráfica
    with open(filepath, encoding='utf-8') as f:
        template = json.load(f)

    if filepath == 'apl_openha.json':
        # Localiza os textos dinâmicos do APL 
        template['mainTemplate']['items'][0]['items'][2]['text'] = globals().get("echo_screen_welcome_text")
        template['mainTemplate']['items'][0]['items'][3]['text'] = globals().get("echo_screen_click_text")
        template['mainTemplate']['items'][0]['items'][4]['onPress']['source'] = get_hadash_url()
        template['mainTemplate']['items'][0]['items'][4]['item']['text'] = globals().get("echo_screen_button_text")

    return template

# Abre dashboard do Home Assistant no navegador Silk
def open_page(handler_input):
    if is_apl_supported:
        # Renderizar modelo vazio, necessário para o comando OpenURL
        # https://amazon.developer.forums.answerhub.com/questions/220506/alexa-open-a-browser.html
        
        handler_input.response_builder.add_directive(
            RenderDocumentDirective(
                token=apl_document_token,
                document=load_template("apl_empty.json")
            )
        )
        
        # Open default page of dashboard
        handler_input.response_builder.add_directive(
            ExecuteCommandsDirective(
                token=apl_document_token,
                commands=[OpenUrlCommand(source=get_hadash_url())]
            )
        )

# Monta a URL do dashboard do Home Assistant
def get_hadash_url():
    ha_dashboard_url = globals().get("home_assistant_url")
    ha_dashboard_url += "/{}".format(globals().get("home_assistant_dashboard", "lovelace"))
    
    # Adicionando o modo kioskmode, se estiver ativado
    home_assistant_kioskmode = bool(globals().get("home_assistant_kioskmode", False))
    if home_assistant_kioskmode:
        ha_dashboard_url += '?kiosk'
    
    logger.debug(f"ha_dashboard_url: {ha_dashboard_url}")
    return ha_dashboard_url

class HelpIntentHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return ask_utils.is_intent_name("AMAZON.HelpIntent")(handler_input)

    def handle(self, handler_input):
        speak_output = globals().get("alexa_speak_help")
        return handler_input.response_builder.speak(speak_output).ask(speak_output).response

class CancelOrStopIntentHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return ask_utils.is_intent_name("AMAZON.CancelIntent")(handler_input) or ask_utils.is_intent_name("AMAZON.StopIntent")(handler_input)

    def handle(self, handler_input):
        speak_output = random.choice(globals().get("alexa_speak_exit").split(";"))
        return handler_input.response_builder.speak(speak_output).set_should_end_session(True).response

class SessionEndedRequestHandler(AbstractRequestHandler):
    def can_handle(self, handler_input):
        return ask_utils.is_request_type("SessionEndedRequest")(handler_input)

    def handle(self, handler_input):
        return handler_input.response_builder.response

class CatchAllExceptionHandler(AbstractExceptionHandler):
    def can_handle(self, handler_input, exception):
        return True

    def handle(self, handler_input, exception):
        logger.error(exception, exc_info=True)
        speak_output = globals().get("alexa_speak_error")
        return handler_input.response_builder.speak(speak_output).ask(speak_output).response

sb = SkillBuilder()
sb.add_request_handler(LaunchRequestHandler())
sb.add_request_handler(GptQueryIntentHandler())
sb.add_request_handler(HelpIntentHandler())
sb.add_request_handler(CancelOrStopIntentHandler())
sb.add_request_handler(SessionEndedRequestHandler())
sb.add_exception_handler(CatchAllExceptionHandler())

lambda_handler = sb.lambda_handler()
```
---
You will then need to save and deploy the code. Once it has deployed head over to the build tab and click 'Build Skill'
## 2. Update `config.cfg`

Add the following line to your `config.cfg` file:

```
assist_input_entity = input_text.assistant_input
```

---

## 3. Create a Text Helper in Home Assistant

1. Open Home Assistant.
2. Go to:  
   **Settings → Devices & Services → Helpers**
3. Click **Create Helper** → Choose **Text**.
4. Set the following options:
   - **Name:** `assistant_input`
   - **Maximum number of characters:** `255` (this is the hard limit)
5. Click **Create**.

> ⚠️ Note: 255 characters is a hard limitation for prompt size. There's no reliable workaround yet, except embedding other text inputs into the prompt.

---

## 4. Create a Script in Home Assistant

### Step-by-step Instructions:

1. Go to the [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask).
2. On your skill’s home page, click **Copy Skill ID**.
3. In Home Assistant, navigate to **Settings → Automations & Scenes → Scripts**.
4. Click **Add Script** and enter a name like `Prompt Alexa Device`.
5. Click the three-dot menu (⋮) and switch to **YAML mode**.
6. Paste the following YAML into the editor.  
   Replace the placeholders:

   - `*your Skill ID*` → your actual Alexa skill ID  
   - `*the alexa you want to target*` → the `media_player` entity ID of your Alexa device

```
sequence:
  - action: input_text.set_value
    metadata: {}
    data:
      value: "{{prompt}}"
    target:
      entity_id: input_text.assistant_input
  - action: media_player.play_media
    data:
      media_content_id: *your Skill ID*
      media_content_type: skill
    target:
      entity_id: *the alexa you want to target*
  - delay:
      hours: 0
      minutes: 0
      seconds: 10
      milliseconds: 0
  - action: input_text.set_value
    metadata: {}
    data:
      value: none
    target:
      entity_id: input_text.assistant_input 
alias: prompt on Alexa device
description: ""
fields:
  prompt:
    selector:
      text: null
    name: prompt
    description: >-
      The prompt to pass to the skill, used as the first message to start a conversation.
    required: true
```

7. Click **Save**.

---

## 5. Call the Script from an Automation

Now that the script is set up, you can trigger it from an automation. This will:

- Pass a prompt to your Alexa skill
- Begin a spoken conversation using the assistant's response

### Example Automation Action

```
action: script.prompt_alexa_device
metadata: {}
data:
  prompt: >-
    I am cooking in the kitchen, can you offer to play some music,
    suggest a genre based on the time of day and day of the week.
```

> ⚠️ **Important:** Prompts must be **fewer than 255 characters**, or the call will fail.

---
