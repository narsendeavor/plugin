<div align="center">
  <img src="https://imgbs.com/uploads/preview-plugin-d9fa8217.png" alt="NARAKU • TOOLBOX Preview" width="100%"/>

  # NARS'S ENDEAVOR

  [![Roblox Studio Plugin](https://img.shields.io/badge/Roblox-Studio%20Plugin-blue?style=for-the-badge&logo=roblox)](https://www.roblox.com)
  [![Lua](https://img.shields.io/badge/Language-Lua-000080?style=for-the-badge&logo=lua)](https://www.lua.org)
  [![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](#)
  [![WhatsApp Channel](https://img.shields.io/badge/WhatsApp-Join%20Channel-25D366?style=for-the-badge&logo=whatsapp&logoColor=white)](https://whatsapp.com/channel/0029VbDPKtaJuyALlMJKuC2A)

  An all-in-one Roblox Studio plugin and utility suite engineered to streamline development workflows, empower builders, and enhance productivity.
</div>

---

## OVERVIEW

**Nars's Endeavor** provides a comprehensive set of development tools directly within your Roblox environment. From instant asset searching to advanced geometry tools, it condenses essential developer utilities into a single, intuitive interface.

---

## FEATURES

- <img src="https://api.iconify.design/lucide:shopping-bag.svg?color=%23ffffff" width="16" height="16"/> **Toolbox Store**: Direct in-studio search and instant injection for Models, Decals, and Audio with ID saving options.
- <img src="https://api.iconify.design/lucide:compass.svg?color=%23ffffff" width="16" height="16"/> **Archimedes Tools**: Precision axis transformation ($X, Y, Z, X^2, Y^2, Z^2$), custom angles, and rapid curve rendering.
- <img src="https://api.iconify.design/lucide:music.svg?color=%23ffffff" width="16" height="16"/> **Audio Player**: Built-in audio player with real-time progress controls for rapid asset testing.
- <img src="https://api.iconify.design/lucide:wrench.svg?color=%23ffffff" width="16" height="16"/> **Utility Suite**: Integrated developer shortcuts featuring Terrain tools, Asset Importer, and Fly GUI.

---

## API DOCUMENTATION

### Roblox Toolbox Service API

The plugin communicates with the official Roblox API endpoint to handle asset searches dynamically within the studio environment.

**API Endpoint**

`https://apis.roblox.com/toolbox-service/v2/assets:search`

**Request Parameters**

| Parameter | Description |
| :--- | :--- |
| `searchCategoryType` | Specifies the asset category to search |
| `query` | Specifies the search keyword |

**Supported Asset Types**

- Model
- Decal
- Audio
- Plugin

**Category Parameter**

`searchCategoryType` is used to specify the type of asset being searched.

**Response**

The API returns search results via the `creatorStoreAssets[]` array.

**Response Fields**

| Field | Description |
| :--- | :--- |
| `asset.name` | Name of the asset |
| `asset.id` | Unique ID of the asset |
| `creator.name` | Name of the asset creator |

---

## COMMUNITY & UPDATES

Stay updated with the latest news, announcements, and features by joining our official WhatsApp channel:

<div align="center">
  <a href="https://whatsapp.com/channel/0029VbDPKtaJuyALlMJKuC2A">
    <img src="https://img.shields.io/badge/Join_WhatsApp_Channel-25D366?style=for-the-badge&logo=whatsapp&logoColor=white" alt="Join WhatsApp Channel"/>
  </a>
</div>

---

## MAP PREVIEW

Experience the environment and test the tools directly in the interactive map environment:

<div align="center">
  <a href="https://www.roblox.com/id/games/10959918411/Studio-Lite">
    <img src="https://files.catbox.moe/e25apk.webp" alt="Studio Lite Map Preview" width="85%" style="border-radius: 8px;"/>
  </a>

  <br/><br/>

  [![Play Studio Lite](https://img.shields.io/badge/Play_Map-Studio_Lite-00A2FF?style=for-the-badge&logo=roblox&logoColor=white)](https://www.roblox.com/id/games/10959918411/Studio-Lite)
</div>

---

## EXECUTING

Run the loader script in your execution environment:

```lua
loadstring(game:HttpGet("https://raw.githubusercontent.com/narsendeavor/plugin/refs/heads/main/loader.lua"))()
