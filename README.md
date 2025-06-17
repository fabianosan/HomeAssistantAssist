# Home Assistant Assist Alexa Skill (Alexa Hosted)

Alexa Skill that integrates Home Assistant Assist or your preferred Generative AI via the conversation API and also allows you to open your favorite dashboard on Echo Show

---

_Note: This project is still in a very early alpha phase, this means not all features are fully functional yet and
features or usage can change significantly between releases._

### Table of Contents

1. [About](#about)
2. [Features](#features)
3. [Installation](#installation)
4. [Supported languages](#supported-languages)

## About

This is a Alexa skill model that integrates [Home Assistant Assist](https://www.home-assistant.io/voice_control) or your preferred Generative AI through the [conversation API](https://developers.home-assistant.io/docs/intent_conversation_api) and also allows you to open your favorite dashboard on Echo Show devices.

_Note: It's important to highlight that a skill (Alexa Hosted) has an 8-second limit for external requests and this cannot be changed.
This means that your Home Assistant instance needs to respond within 6 or 7 seconds for the skill to work correctly. In other words, very complex AIs or slow models will not work with this skill. It is important to test the agent (Assist) that you intend to use beforehand, with simple and complex commands to ensure that the assistant returns within the expected time._

For slower AIs or servers, try the [AWS Hosted](https://github.com/fabianosan/HomeAssistantAssistAWS) version.

## Features

- Voice command:
    - Interact with [Home Assistant Assist](https://www.home-assistant.io/voice_control)
    - Interact with [Open AI](https://www.home-assistant.io/integrations/openai_conversation) integration
    - Interact with [Extended Open AI](https://github.com/jekalmin/extended_openai_conversation) integration
    - Interact with [Google Generative AI](https://www.home-assistant.io/integrations/google_generative_ai_conversation) integration
- Open Home Assistant dashboard:
    - Say 'open home assistant' or 'open dashboard' to open your prefered dashboard in Home Assistant.
    - Or clique on sceen to do the same.
- Other:
    - Multi-language support (see [Supported languages](#supported-languages))

If you have a feature idea, open a issue to suggest your idea for implementation.

## Installation

For instructions how to set this skill up refer to the [installation](doc/en/INSTALLATION.md) or [update](doc/en/UPDATE.md) page.
For instruction on how to update skill to add conversation functinality initiated by a prompt in home assistant, and how to set up related home assistant automations please refer to [conversation starter installation](doc/en/installation_updated_for_conversation_starter.md).

## Supported languages

The skill has support for the following languages:

- German (Germany)
- English (Australia)
- English (Canada)
- English (England)
- English (United States)
- Dutch (Netherlands)
- Spanish (Spain)
- Spanish (Mexico)
- Spanish (United States)
- French (Canada)
- French (France)
- Italian (Italy)
- Portuguese (Brazil)
- Portuguese (Portugal)

Note: If your language is not supported, please open an `issue` attaching your own translated version of the file [en-US.lang](lambda/locale/en-US.lang). Only languages supported by Alexa framework will be added (please, check this link: https://developer.amazon.com/en-US/docs/alexa/device-apis/list-of-interfaces.html)

---



# Home Assistant Assist Alexa Skill (Alexa Hosted)

Skill Alexa que integra o Home Assistant Assist ou sua IA Generativa preferida via a API de conversação e também permite abrir seu painel favorito no Echo Show

---

_Nota: Este projeto ainda está em uma fase alfa muito inicial, o que significa que nem todos os recursos estão totalmente funcionais e os recursos ou o uso podem mudar significativamente entre as versões._

### Índice

1. [Sobre](#sobre)
2. [Recursos](#recursos)
3. [Instalação](#instalação)
4. [Idiomas suportados](#idiomas-suportados)

## Sobre

Este é um modelo de skill Alexa que integra o [Home Assistant Assist](https://www.home-assistant.io/voice_control) ou sua IA Generativa preferida através da [API de conversação](https://developers.home-assistant.io/docs/intent_conversation_api) e também permite abrir seu painel favorito em dispositivos Echo Show.

_Observação: É importante destacar que uma skill (Alexa Hosted) possui um limite de 8 segundos para requisições externas e isso não pode ser alterado.
Isso significa que sua instância do Home Assistant precisa responder em até 6 ou 7 segundos para a skill funcionar corretamente, ou seja, IA's muito complexas, ou modelos lentos não vão funcionar com essa skill, importante testar o agente (Assist) que pretende utilizar antes, com comandos simples e complexos para garantir que o assistente retorna dentro do tempo esperado._

Para IA's ou servidores mais lentos, tente a versão [AWS Hosted](https://github.com/fabianosan/HomeAssistantAssistAWS).

## Recursos

- Comando de voz:
    - Interagir com o [Home Assistant Assist](https://www.home-assistant.io/voice_control)
    - Interagir com a integração [Open AI](https://www.home-assistant.io/integrations/openai_conversation)
    - Interagir com a integração [Extended Open AI](https://github.com/jekalmin/extended_openai_conversation)
    - Interagir com a integração [Google Generative AI](https://www.home-assistant.io/integrations/google_generative_ai_conversation)
- Abrir painel do Home Assistant:
    - Diga 'abrir home assistant' ou 'abrir painel' para abrir seu painel preferido no Home Assistant.
    - Ou clique na tela para fazer o mesmo.
- Outros:
    - Suporte a vários idiomas (veja [Idiomas suportados](#idiomas-suportados))

Se você tiver uma ideia de recurso, abra um issue para sugerir sua ideia para implementação.

## Instalação

Para obter instruções sobre como configurar essa skill, consulte a página de [instalação](doc/pt/INSTALLATION.md) ou [atualização](doc/pt/UPDATE.md).

## Idiomas suportados

A skill tem suporte para os seguintes idiomas:

- Alemão (Alemanha)
- Inglês (Austrália)
- Inglês (Canadá)
- Inglês (Inglaterra)
- Inglês (Estados Unidos)
- Espanhol (Espanha)
- Espanhol (México)
- Espanhol (Estados Unidos)
- Francês (Canadá)
- Francês (França)
- Italiano (Itália)
- Holandês (Holanda)
- Português (Brasil)
- Português (Portugal)
