---
layout: post
title:  "Ramalama - prostředí pro lokální LLM"
author: "Adam Hošek"
comments: true
tags: Ramalama localLLM
excerpt_separator: <!--more-->
sticky: false
hidden: false
---

Letos v lednu jsem začal řešit, jak bych mohl zlepšít svoji produktivitu pomocí jazykových modelů. Pracuji jako programátor a zde se jejich využití vyloženě nabízí. Již v minulosti jsem experimentoval a používal nástoje jako chatGPT od OpenAI a Claude od Anthropicu, které jsou bezpochyby perfektní, snadno dostupné a dost dobře použitelné i v bezplatné verzi. Velice jsem si je oblíbil v momentech, kdy namísto hledání v dokumetaci, použiji tohoto asistenta a mám vyhráno. Originální dokumentaci potom mohu použít pro ověření a rozvinutí řešení. Vzít ale asistenta, integrovat jej do vývojového prostředí, předhodit celou codebase vzdáleně běžícímu modelu a pracovat nad tím, no... Zde jsem měl obavy o soukromí, variantu použití GitHub Copilot atd. jsem ted zavrhl.
<!--more-->
Rád bych však používal LLM přímo v návaznosti k aktuálnímu kódu, ideálně integrované přímo do vývojového prostředí, což je v mém případě VS Code. Shodou okolností jsem narazil na rozšíření [`vscode-paver`](https://github.com/redhat-developer/vscode-paver), což je, jak jsem později zjistil, nadstavba nad [`continue`](https://www.continue.dev/). Paver ovšem vyžaduje použítí [`ollama`](https://ollama.com/) ke spouštění modelu, což je (pokud se nic nezměnilo) nástroj pro spouštění modelů, který není zcela open-source a jeho nastavení pro správné využití hardwaru nebylo zcela uživatelsky přívětivé. Samotné rozšíření continue však vypadalo použitelně s možností vlastní konfigurace.

Netrvalo dlouho a objevil jsem nástroj pro spouštění modelů [`ramalama`](https://github.com/containers/ramalama) za kterým stojí vývojáři z Red Hatu (kde shodou okolností pracuji). Ramalama si klade za cíl snadné a bezpečné spouštění jazykových modelů kompletně v kontejnerizovaném prostředí - tedy něco, co jsem přesně hledal. Volba byla tím pádem jasná jasná, `continue` ve spojení s `ramalama`.

## Konfigurace `ramalama`
Začněme s konfigurací ramalamy. Provozuji Fedoru 41 na AMD Ryzen 5700X s 64 GB ram a grafickou kartou Radeon 6700 XT 12 GB. Instalaci provedeme buď pomocí `pip` nebo stažením `python3-ramalama` z repozitářů Fedory. Další možnosti v sekci [instalace](https://github.com/containers/ramalama?tab=readme-ov-file#install).

Spuštěním `ramalama pull <název-modelu>`, tedy například `ramalama pull granite-code:8b` stáhneme model z repozitáře. Výchozí repozitář pro stahování je [ollama.com](https://ollama.com/library), ovšem je dostupný také [`Hugging Face`](https://huggingface.co/) a `OCI Container registers` jakožto například [quay.io](https://quay.io/). Přepínat mezi nimi lze pomocí proměnné [export RAMALAMA_TRANSPORT=huggingface](https://github.com/containers/ramalama?tab=readme-ov-file#transports).

Máme tedy stažený model, aby nebylo nutné jej pokaždé spouštět celým názvem, je výhodné si nastavit zkratky v `~/.config/ramalama/shortnames.conf` například jako:
```
[shortnames]
  "granite-code:8b" = "huggingface://ibm-granite/granite-8b-code-base-4k-GGUF/granite-8b-code-base.Q4_K_M.gguf"
  "qwen-coder:14b" = "ollama://qwen2.5-coder:14b"
```

Vzhledem k tomu, že používám grafickou kartu, která není ve výchozím stavu podporovaná pro běh modelu, musím nastavit proměnnou `export HSA_OVERRIDE_GFX_VERSION=10.3.0` (osobně jsem si ji umístil do `~/.bashrc`). Seznam podporovaných `ROCm` grafik je dostupný na githubu [ollama](https://github.com/ollama/ollama/blob/main/docs/gpu.md) stejně jako [hodnoty pro nastavení](https://github.com/ollama/ollama/blob/main/docs/gpu.md#overrides-on-linux) `HSA_OVERRIDE_GFX_VERSION` - vezmeme hodnutu z tabulky a mapujeme ji jako `x.y.z`, např. `gfx1030` -> `10.3.0` (jak je ostatně uvedeno v popisu).

Model následně spustíme pomocí například `ramalama --gpu serve granite-code:8b --name granite --detach -p 12400`, kde:
- přepínač `gpu` zajistí spuštění na grafické kartě, při vynechání se model ve výchozím nastavení provozuje na procesoru
- `serve` spustí model v režimu, kdy je dostupný jako API či případně přes web `127.0.0.0:12400`, jinak běží jednoduchý chat v příkazové řádce
- `name` je název spuštěného kontejneru, dostupné posléze pomocí `ramalama ps` (vylistování běžících) či `ramalama stop granite` (ukončení modelu)
- `detach` "nechceme vidět výstup z kontejneru"
- `p` port, na kterém model spustíme, lze zvolit jakýkoli, jen je vhodné se vyvarovat [rezervovaným](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers)
Či jen pouhého `ramalama run granite` pro spuštění na procesoru v příkazové řádce. Po spuštění pomocí `serve` jsme schopni navštívit `127.0.0.1:12400` ve webovém prohlížeči, kde vidíme základní rozhraní chatu.

### Výběr modelu
Výběr modelu záleží na osobních preferencích. Základní informace platné pro všechny modely:
- Množství parametrů, udávané typicky jako 14b, 8b atd. určuje množství znalostí a jeho komplexnost.
- Kvantizace, udavané jako Q4 určuje, jak přesně jsou informace v modelu uloženy. Čím vyšší, tím přesnější uložení.
- MOE (mixture of experts) - velký model, ze kterého se vybere jen menší část, je tudíž méně náročný na paměť

## Konfigurace `continue`
Continue nainstalujeme jako [rozšíření](https://marketplace.visualstudio.com/items?itemName=Continue.continue) pro Visual Studio Code. Konfiguraci provedeme v `~/.continue/config.json (dříve byl tento soubor dostupný přes proklik přímo ve VS Code). Moje experimentální konfigurace vypadá takto následovně:
```
{
  "models": [
    {
      "model": "granite-code",
      "title": "Granite Code Local LLM",
      "provider": "llama.cpp",
      "contextLength": 4192,
      "completionOptions": {
        "temperature": 0.7,
        "topP": 0.9,
        "topK": 40,
        "presencePenalty": 0.1,
        "frequencyPenalty": 0.2,
        "maxTokens": 1500
      },
      "stop": [
        "System:",
        "Question:",
        "Answer:",
        "<|endoftext|>",
        "<|im_start|>",
        "im_end"
    ],

      "systemMessage": "You are Granite, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behavior. You are an intelligent AI programming assistant, utilizing a Granite code language model developed by IBM. Your primary function is to assist users in programming tasks, including code generation, code explanation, code fixing, generating unit tests, generating documentation, application modernization, vulnerability detection, function calling, code translation, and all sorts of other software engineering tasks. Every code you write is completely formatted into codeblock and properly commented. When asked to fix code, output only the corrected part of the code, this means diff.",
      "apiBase": "http://0.0.0.0:12400"
    },
    {
      "model": "qwen2.5-coder:14b",
      "title": "qwen Local LLM",
      "provider": "llama.cpp",
      "contextLength": 8192,
      "completionOptions": {
        "temperature": 0.7,
        "topP": 0.9,
        "topK": 40,
        "presencePenalty": 0.1,
        "frequencyPenalty": 0.2,
        "maxTokens": 4192
      },
      "systemMessage": "You are deepseek. You are a cautious assistant, programming expert with a lot of experience in field. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behavior. You are an intelligent AI programming assistant. Your primary function is to assist users in programming tasks, including code generation, code explanation, code fixing, generating unit tests, generating documentation, application modernization, vulnerability detection, function calling, code translation, and all sorts of other software engineering tasks. Every code you write is completely formatted into codeblock and properly commented. When asked to fix code, output only the corrected part of the code, this means diff.",
      "apiBase": "http://0.0.0.0:12401"
    }
  ],
  // removed the rest which was pre-generated
}
```
Hodnoty jsou určeny primárně experimentováním, co pro mě fungovalo nejlépe. Zajímavé hodnoty k nastevení/experimentům s hodnotami:
- `contextLength` velikost kontextového okna, zaleží také na dostupné paměti
- `temperature` <0; 1> čím blíže k 1 tím více "kreativní", 0 přesnější
- `topP` vybírá nejmenší množinu tokenů, jejichž kumulativní pravděpodobnost dosahuje hodnoty p (hodnota mezi 0 a 1); vyšší p umožní širší spektrum tokenů (větší rozmanitost), zatímco nižší p omezuje výstup na velmi pravděpodobné tokeny (konzervativnější odpovědi) *(výstup z chatGPT který jsem k experimentům použil)*
- `topK` omezuje výběr modelu na k nejpravděpodobnějších tokenů; zvýšení hodnoty k umožňuje širší výběr a tím vyšší rozmanitost, zatímco snížení hodnoty vede k determinističtějším a konzervativnějším odpovědím *(výstup z chatGPT který jsem k experimentům použil)*
- `presencePenalty` penalizuje výběr tokenů, které se již v textu objevily, bez ohledu na jejich četnost; zvýšení tohoto parametru podporuje generování nových tokenů a snižuje opakování, zatímco snížení může vést k častějšímu opakování již použitých tokenů *(výstup z chatGPT který jsem k experimentům použil)*
- `frequencyPenalty` snižuje pravděpodobnost opětovného výběru tokenů úměrně jejich četnosti v dosud vygenerovaném textu; vyšší hodnota tohoto parametru dále omezuje opakování slov, zatímco nižší hodnota může vést k většímu opakování *(výstup z chatGPT který jsem k experimentům použil)*
- `maxTokens` maximální délka odpovědi
- `systemMessage` tuto zprávu model dostane jako začátek konverzace. V případě krátkých dotazů se mi stávalo, že model reagoval přímo na tuto zprávu.

Následně vidíme v `continue` možnost na přepínání modelů pod chatem, moje nejčastější užití je označení požadované sekce/sekcí v kódu, přidání do kontextu pomocí `CTRL+L` a diskuze nad problémem. Inline doplňování se mi příliš neosvědčilo a přijde mi, že moji práci spíše zpomaluje.

## Zhodnocení
Tuto konfiguraci aktuálně používám a výsledky jsou použitelné, místy je nutné model restartovat z důvodu generování náhodných znaků a požadavek zopakovat. Možnosti `continue` jsou mnohem bohatší, toto je vícemnéně základ použití. Nejlepších výsledků jsem na své konfiguraci dosahoval s modelem `qwen-coder:14b`, který je dostatečně svižný s dobrými výsledky. Model `granie-code` by pravděpodobně byl také použitelný, avšak v 8b variantě byl nedostatečný a 20b byla zase příliš pomalá, jelikož se nevešel celý do VRAM. `Deepseek-r1` v 14b variantě je zase příliš přemýšlivý, což je ale jeho účel, vzhledem k tomu, že jde o reasoning model ;). 

Je také potřeba mít na paměti, že i když model běží lokálně a naše data neopustí počítač, odpovědi jsou stále generované na základě dat poskytnutých k učení. Je otázkou, jaká tyto data měla licenci a použití v produkčním kódu je z hlediska licenčních práv komplikované. Použití tohoto nástroje jako partnera při řešení problémů (ne pro konkrétní implementaci) by ale mělo být v pořádku.

V experimentování však i nadále pokračuji.

## Cheat sheet
### Install:
https://github.com/containers/ramalama

- download the model, e.g.: `ramalama pull granite-code:20b`
NOTE: Name of the model is got from ollama.com/library/

### Set shortnames

- set the short name in `~/.config/ramalama/shortnames.conf` like:
```
[shortnames]
   "granite-code:20b" = "huggingface://ibm-granite/granite-20b-code-base-8k-GGUF/granite-20b-code-base.Q4_K_M.gguf"
```

### Run the model:
- set the env var for amd rx6700 xt: `export HSA_OVERRIDE_GFX_VERSION=10.3.0`
- run the model: `ramalama --gpu run granite-code:20b` 

### Run for Continue
- `ramalama --gpu serve granite-code:8b --temp 0.95 --ctx-size 6000 --name granite --detach -p 12400`
- `ramalama --gpu serve qwen-coder:14b --temp 0.95 --ctx-size 9000 --name qwen --detach -p 12401`
