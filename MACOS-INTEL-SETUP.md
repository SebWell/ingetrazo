# IngeTrazo sur Mac Intel (macOS 12) — notes d'installation

Ces notes documentent l'installation d'IngeTrazo **depuis les sources** sur un
MacBook **Intel** (macOS 12.7.6 Monterey), où le `.dmg` officiel (Apple
Silicon uniquement) ne s'exécute pas. Elles couvrent aussi le branchement
d'**OpenCode** : le fournisseur **OpenCode Go** dans l'assistant interne et le
**serveur MCP** d'IngeTrazo.

> Ces notes sont personnelles (elles vivent dans le fork). Le code, lui, est
> destiné à l'amont (voir la PR).

## 1. Python et environnement virtuel

Le venv est créé avec `uv` (pas de Homebrew, pas de droits admin).

```bash
cd ~/Documents/Developer/ingetrazo
uv python install 3.12
uv venv --python 3.12 venv
uv pip install --python venv/bin/python -r requirements.txt
```

**Épingler PySide6** : les 6.10+ exigent macOS 13/14 ; sur Monterey il faut la
dernière 6.9.x.

```bash
uv pip install --python venv/bin/python 'PySide6==6.9.3'
```

Contrôle d'intégrité :

```bash
./venv/bin/python main.py --check     # doit finir par « OK »
```

Lancer depuis les sources :

```bash
./venv/bin/python main.py
```

## 2. Construire l'application `.app`

Le dépôt fournit le script PyInstaller complet :

```bash
./packaging/build-macos-app.sh
```

Il produit :

- `dist/IngeTrazo.app` (autonome, ~213 Mo, x86_64)
- `dist/IngeTrazo-<version>-macos-x86_64.dmg`

Installer l'app :

```bash
rm -rf ~/Applications/IngeTrazo.app
mv dist/IngeTrazo.app ~/Applications/
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -f ~/Applications/IngeTrazo.app
```

> Le venv n'a pas `pip` (créé par `uv`) : si un outil réclame `pip`, utiliser
> `uv pip install --python venv/bin/python <paquet>`.

## 3. Fournisseur « OpenCode Go » dans l'assistant interne

L'assistant IA intégré (onglet **IA**) détecte le fournisseur par préfixe de
clé. OpenCode Go n'est pas reconnu par préfixe : il faut le **sélectionner
explicitement**.

1. Onglet **IA**.
2. **Provider** : **OpenCode Go** (pas « Auto »).
3. **Key** : clé API OpenCode Go (compte OpenCode Go).
4. **Model** : `deepseek-v4.1-flash` (défaut), ou un autre modèle du tableau
   ci-dessous.
5. **Test connection** → « OK ».

Côté code (`core/ai.py`), la passerelle OpenCode Go est compatible OpenAI
(`https://opencode.ai/zen/go/v1`) **mais** :

- elle exige un en-tête de session `x-opencode-session` (sinon `HTTP 400
  Missing sessionID`), envoyé par `build_request`/`chat` (ID stable par
  conversation côté `plugins/ai_assistant.py`) ;
- elle sert **chaque famille de modèles sur un endpoint différent**, choisi
  par `_wire_style()` :

| Famille (préfixe du modèle) | Endpoint | Auth |
|---|---|---|
| DeepSeek, GLM, Kimi, MiMo, Hy, Space Bunny, LongCat, Step 5 | `/chat/completions` (OpenAI) | `Bearer` |
| Claude, Qwen, MiniMax | `/messages` (Anthropic) | `x-api-key` |
| Grok, GPT Luna, Muse Spark | `/responses` (OpenAI Responses) | `Bearer` |

**Vision** : pour OpenCode Go, les familles `claude`, `gpt`, `grok`,
`gemini`, `qwen` (et tout id contenant `vision`, p. ex.
`deepseek-v4-flash-vision-exp`) sont traitées comme multimodales : cocher
« Envoyer les captures d'écran » fait alors parvenir le viewport au modèle.

## 4. Connecter OpenCode au serveur MCP d'IngeTrazo

IngeTrazo expose un serveur MCP stdio qui pilote le document **ouvert** via le
plugin « AI Bridge ». Le fichier `opencode.jsonc` à la racine du projet le
branche à OpenCode (et fixe le modèle par défaut) :

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode-go/deepseek-v4.1-flash",
  "mcp": { "servers": { "ingetrazo": {
    "type": "local",
    "command": ["python3", "scripts/ingetrazo_mcp.py"]
  } } }
}
```

Dans IngeTrazo, **allumer le pont** : onglet **IA** ▸ section « Puente IA
(MCP) » ▸ **Encender puente** (ou **Extensiones ▸ Puente IA (MCP)**). Il
écoute alors sur `127.0.0.1:4763`.

Vérifier côté OpenCode (lancé depuis ce dossier) :

```bash
opencode mcp list        # → ✓ ingetrazo  connected
```

Outils exposés à l'agent : `run_python`, `query_model`, `screenshot`, `undo`,
`redo` — chaque appel est **un pas d'undo** dans le document.

## 5. Divers

- **DWG** : non supporté sur macOS (le `dwg2dxf` fourni dans git est un binaire
  Linux). `--check` le signale désormais « MISSING » (au lieu de « found »).
- **Mise à jour** : `git pull` puis refaire `uv pip install` et
  `./packaging/build-macos-app.sh`.
- **Gatekeeper** : app signée ad-hoc ; sur une autre machine, premier lancement
  par **clic droit ▸ Ouvrir**.
