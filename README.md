# Discord Quest Completer

![Discord Quest Completer](https://img.shields.io/badge/Project-Discord%20Quest%20Completer-7289DA?style=flat&logo=discord)
![Platform-Windows](https://img.shields.io/badge/Platform-Windows-0078D7?style=flat&logo=windows)
![Tech-Rust](https://img.shields.io/badge/Tech-Rust-000000?style=flat&logo=rust)
![Tech-Vue.js](https://img.shields.io/badge/Tech-Vue.js-3EAF7C?style=flat&logo=vue.js)

> Lightweight Windows desktop app to complete Discord Quests and simulate verified Discord game status without installing full games.

---

## ✨ Why this exists

This project helps you trigger Discord Rich Presence and complete supported Discord Quests by launching tiny verified game simulators instead of the full application.

It is built to be:
- fast to run
- small in disk footprint
- tailored for Windows and Discord Verified games
- easy to build with Tauri, Rust, and Vue

---

## 🚀 What it does

- Simulates a Discord Verified game process
- Triggers Discord game detection and Rich Presence updates
- Lets you complete Quest requirements without heavy installs
- Keeps the app lightweight and portable

---

## 🧩 Features

- ✅ Verified Discord game detection support
- ✅ Minimal game runner executable
- ✅ Rich Presence-compatible simulation
- ✅ Designed for Windows desktop use
- ✅ Built with Rust + Vue + Tauri

---

## 📦 Repository Structure

```text
DC/
├── src/                # Vue frontend
├── src-tauri/          # Tauri app configuration and Rust backend
├── src-win/            # Windows runner app
├── public/             # Static assets
├── package.json        # Frontend scripts and dependencies
├── tauri.conf.json     # Tauri application config
└── README.md           # This file
```

---

## 🛠️ Tech Stack

- **Rust** for native Windows runner code
- **Vue 3** for frontend UI
- **Tauri** for desktop app packaging
- **Tailwind CSS** for styling
- **pnpm** for package management

---

## 🏁 Quick Start

### Install dependencies

```bash
pnpm install
```

### Build runner and copy resources

```bash
pnpm build:runner:win
pnpm copy:runner:win
```

### Start development mode

```bash
pnpm tauri dev
```

---

## ⚙️ Notes

- WebView2 is required to run the Tauri app on Windows.
- The runner is built as a tiny executable that mimics Discord game process detection.
- This repo currently targets Windows only.

---

## 🎯 Goals

- Improve Discord Quest compatibility
- Reduce runner binary size
- Keep the app stable and easy to maintain
- Add richer activity simulation features over time

---

## 💡 Contributing

Contributions are welcome! If you want to help:

- open issues for bugs or ideas
- send pull requests with fixes or improvements
- suggest better Discord detection flow

---

## 📬 About

Built by **theycallmearnav**.

Love clean tools, small utilities, and elegant desktop workflows.

---

## 🔗 Useful links

- GitHub: https://github.com/theycallmearnav
- This repo: (https://github.com/theycallmearnav/Discord-Quest-Completor)

---

## 📌 License

This repo includes a `LICENSE` file. Review it for details.


### The Discord's RPC server and Rich Presence 

There is also an experimental action like Discord RPC functionality along the selected game.
It connects to Discord's RPC Gateway to send Activity updates for the selected game using its App ID, even if the game is not actually running. (This is Rich Presence only so Quests will not detect it)

(This may not work for some time, as Discord updates their RPC and SDK. The syntax I used in the Rust code may not be updated, as it is not one of the main focus of this app.)

Though this is functional as it uses Discord Rich Presence, what happens is it uses the App ID of some App on Discord and connects it to the RPC.  
See: [Discord Developer Docs – Creating an App](https://discord.com/developers/docs/quick-start/getting-started#step-1-creating-an-app)

For example, the App ID for Overwatch is `356875221078245376`, and we use it with something like [discordjs/rpc](https://github.com/discordjs/RPC).


```js
// Set this to your Client ID.
const clientId = '356875221078245376'; // This is Overwatch's App ID on Discord, not one I created.
DiscordRPC.register(clientId);

const rpc = new DiscordRPC.Client({ transport: 'ipc' });
const startTimestamp = new Date();

// You will see "Verified Overwatch" on Discord Activity with custom details.
rpc.setActivity({
  details: `Bleet bleet`,
  state: 'in bleet bleet party',
  startTimestamp,
});
```

This may not be the intended use of Discord's RPC and may violate their Terms of Service.  
I am not entirely sure if you can use others' App IDs other than the ones you own in the Discord Developer Dashboard for the application you are developing.
Use this feature at your own risk.


### Disclaimer

This tool is intended for educational purposes and personal use. Please respect Discord's terms of service, partners, game publishers and advertisers rights when using this application.

The creators and maintainers of this project are not liable for any damages, account suspensions, or other consequences that may arise from using this software. Use at your own risk.

Discord is a registered trademark of Discord Inc. It is referenced on this open-source project for descriptive and definition purposes only and does not imply any affiliation, sponsorship, or endorsement by Discord Inc in any way.

---

<!--
## Other Alternatives

If you can't install this application for any reason, there is some steps on a gist from [aamiaa](https://github.com/aamiaa/) that allows you to use Discord client's Web Inspector and paste the code provided and complete the quest.

See the guide here: https://gist.github.com/aamiaa/204cd9d42013ded9faf646fae7f89fbb
-->


## License

[MIT License](LICENSE)© Arnav Badola - 2026

---


[github-badge]: https://img.shields.io/badge/-Github%20Sponsor-fafbfc?logo=GitHub%20Sponsors
[github-sponsors]: https://github.com/sponsors/markterence
[paypal-badge]: https://img.shields.io/badge/-Paypal-002991?logo=Paypal
[paypal]: https://paypal.me/MarkTerenceTiglao

