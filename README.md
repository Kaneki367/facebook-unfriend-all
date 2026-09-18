# FB Bulk Unfriend

A browser console script to bulk-unfriend everyone on Facebook, since Facebook doesn't provide a built-in way to do this. Runs entirely client-side in your own browser — no extension install required, no data sent anywhere.

## ⚠️ Disclaimer

- This is an **unofficial** script that automates clicks on Facebook's web UI. It is **not sanctioned by Meta**, and using it may violate Facebook's Terms of Service.
- Facebook's page structure changes over time, so this script **can break at any point** without warning. If it stops working, the selectors likely need updating to match Facebook's current markup.
- Running bulk automated actions carries a **real risk of temporary rate-limiting or a security checkpoint** on your account. Use at your own discretion and risk.
- This script permanently removes friends one by one. **There is no undo** — you'd need to re-send friend requests individually afterward.
- **This is an all-or-nothing tool.** It removes every friend in your list in order and gives you no option to select or exclude specific people. If you only want to remove some friends, this script is not built for that — you would need to stop it manually partway through, or modify the code yourself to add a skip-list.
- The author is not responsible for any account restrictions, data loss, or other consequences of using this script.

## What it does

- **Removes ALL of your friends indiscriminately.** This script does not let you pick and choose who to keep — it processes every friend in your list one by one, with no filtering, favorites list, or exceptions. If you want to keep certain friends, you must stop the script manually (`stopUnfriend()`) before it reaches them, or remove them from view/access some other way beforehand. There is no built-in way to exclude specific people.
- Automatically finds each friend's "..." menu on your Facebook friends list
- Opens the menu, clicks **Unfriend**, then confirms
- Skips people it's already processed (tracked by name, not DOM element, so it survives Facebook re-rendering the page)
- Skips Facebook's **"People You May Know"** suggestions if they appear near the friends list
- Automatically handles the **"Not Friends Yet"** error popup (which can appear if the UI is showing stale data) by dismissing it and moving to the next person
- Scrolls down automatically only when it runs out of visible friends to process, then waits 10 seconds for more to load before giving up
- Shows a live **progress bar** overlay (bottom-right of the page) tracking how many friends have been removed out of your total friend count
- **Stops automatically** once it reaches your total friend count, or if no more friends are found after scrolling

## How to use

1. Log into Facebook and go to your **Friends** list (`facebook.com/me/friends` or your profile's Friends tab).
2. Open your browser's Developer Tools:
   - **Windows/Linux:** `F12` or `Ctrl+Shift+I`
   - **Mac:** `Cmd+Option+I`
3. Click the **Console** tab.
4. Facebook shows a red "Stop!" self-XSS warning the first time you open the console — this is normal and appears for everyone, regardless of what you're about to paste. Type `allow pasting` into the console and press Enter to get past it.
5. Paste the full script (below) into the console and press Enter.
6. Watch the progress bar in the bottom-right corner of the page. Don't close the tab while it's running.

## Stopping it manually

At any point, run this in the console to halt the script:

```javascript
stopUnfriend()
```

It will report how many friends were removed so far before stopping.

## Adjusting speed

The script currently runs at a fast pace (roughly 1–2 seconds per person). If you notice Facebook throwing checkpoints, CAPTCHAs, or unusual warnings, consider slowing it down by increasing the delay values inside the script:

```javascript
await randomDelay(300, 500);   // after clicking "..." — increase for more caution
await randomDelay(250, 450);   // after clicking "Unfriend" — increase for more caution
function scheduleNext() {
  if (running) randomDelay(400, 800).then(unfriendNext); // gap between each person
}
```

Larger numbers = slower, safer; smaller numbers = faster, riskier.

## The script

```javascript
(function () {
  let count = 0;
  let running = true;
  let totalFriendsNum = null;
  const triedNames = new Set();

  function randomDelay(min, max) {
    return new Promise(r => setTimeout(r, min + Math.random() * (max - min)));
  }

  function parseFriendCount(text) {
    const match = text.match(/([\d.]+)\s*(k|m)?/i);
    if (!match) return null;
    let num = parseFloat(match[1]);
    const suffix = (match[2] || '').toLowerCase();
    if (suffix === 'k') num *= 1000;
    if (suffix === 'm') num *= 1000000;
    return Math.round(num);
  }

  function getTotalFriendsCount() {
    const el = Array.from(document.querySelectorAll('a, span'))
      .find(e => /^\d+(\.\d+)?[km]?\s*friends$/i.test(e.innerText.trim()));
    if (el) return parseFriendCount(el.innerText.trim());
    return null;
  }

  // ---- Status bar overlay ----
  function createStatusBar() {
    const bar = document.createElement('div');
    bar.id = 'fb-unfriend-status';
    bar.style.cssText = `
      position: fixed; bottom: 20px; right: 20px; z-index: 999999;
      background: #1c1e21; color: #fff; padding: 14px 18px;
      border-radius: 10px; font-family: sans-serif; font-size: 13px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.4); width: 260px;
    `;
    bar.innerHTML = `
      <div style="margin-bottom:6px; font-weight:600;">Bulk Unfriend Progress</div>
      <div style="background:#3a3b3c; border-radius:6px; height:10px; overflow:hidden; margin-bottom:6px;">
        <div id="fb-unfriend-fill" style="background:#4bb543; height:100%; width:0%; transition: width 0.3s;"></div>
      </div>
      <div id="fb-unfriend-text">0 / ? unfriended</div>
    `;
    document.body.appendChild(bar);
  }

  function updateStatusBar() {
    const fill = document.getElementById('fb-unfriend-fill');
    const text = document.getElementById('fb-unfriend-text');
    if (!fill || !text) return;
    const pct = totalFriendsNum ? Math.min(100, (count / totalFriendsNum) * 100) : 0;
    fill.style.width = pct + '%';
    text.textContent = `${count} / ${totalFriendsNum || '?'} unfriended`;
  }

  function removeStatusBar(finalMessage) {
    const bar = document.getElementById('fb-unfriend-status');
    if (bar) {
      const text = document.getElementById('fb-unfriend-text');
      if (text) text.textContent = finalMessage;
      setTimeout(() => bar.remove(), 5000);
    }
  }
  // ----------------------------

  function findFriendCardButtons() {
    const candidates = Array.from(document.querySelectorAll('div[role="button"], span[role="button"]'))
      .filter(el => {
        const label = (el.getAttribute('aria-label') || '').toLowerCase();
        const text = el.innerText.trim();
        return text === "..." || text === "•••" || label.includes("see options") || label.includes("more options") || label.includes("for ");
      });

    const results = [];
    for (const btn of candidates) {
      let card = btn.closest('div[class]');
      let hops = 0;
      let name = null;

      // Skip anything inside a "People You May Know" section
      const isInSuggestions = (() => {
        let node = btn;
        let checkHops = 0;
        while (node && checkHops < 15) {
          const t = node.innerText || '';
          if (/people you may know/i.test(t) && t.length < 2000) {
            return true;
          }
          node = node.parentElement;
          checkHops++;
        }
        return false;
      })();

      if (isInSuggestions) continue;

      const label = btn.getAttribute('aria-label') || '';
      const m = label.match(/for (.+)/i);
      if (m) name = m[1].trim();

      while (card && hops < 6 && !name) {
        const nameLink = card.querySelector('a[role="link"]');
        const cardText = card.innerText || '';
        if (nameLink && nameLink.innerText.trim().length > 1 && /mutual friend|friends/i.test(cardText)) {
          name = nameLink.innerText.trim().split('\n')[0];
          break;
        }
        card = card.parentElement;
        hops++;
      }

      if (name && !triedNames.has(name)) {
        results.push({ btn, name });
      }
    }
    return results;
  }

  function findMenuItem(keyword) {
    return Array.from(document.querySelectorAll('div[role="menuitem"], span, div[role="button"]'))
      .find(el => el.innerText.trim().toLowerCase() === keyword);
  }

  function checkAndDismissNotFriendsDialog() {
    const heading = Array.from(document.querySelectorAll('div, span'))
      .find(el => el.innerText.trim() === "Not Friends Yet");
    if (heading) {
      const okBtn = Array.from(document.querySelectorAll('div[role="button"], span'))
        .find(el => el.innerText.trim() === "OK");
      if (okBtn) {
        okBtn.click();
        console.log("⏭️ 'Not Friends Yet' popup — dismissed, moving to next person.");
      }
      return true;
    }
    return false;
  }

  function stopEverything(reason) {
    running = false;
    console.log(`⏹ ${reason} Total unfriended: ${count} / ${totalFriendsNum || '?'}`);
    removeStatusBar(`Done — ${count}/${totalFriendsNum || '?'} unfriended`);
  }

  async function unfriendNext() {
    if (!running) return;

    if (totalFriendsNum && count >= totalFriendsNum) {
      return stopEverything("🎉 Reached total friend count —");
    }

    let candidates = findFriendCardButtons();

    if (candidates.length === 0) {
      console.log("No unprocessed friend cards visible — scrolling down.");
      window.scrollBy(0, 1200);

      await randomDelay(10000, 10000); // wait 10s for new content to load

      candidates = findFriendCardButtons();

      if (candidates.length === 0) {
        return stopEverything("🏁 Still no friends found after scrolling and waiting 10s.");
      }
    }

    const { btn, name } = candidates[0];
    triedNames.add(name);
    console.log(`👤 Processing: ${name}`);

    btn.click();
    await randomDelay(300, 500);

    const unfriendOption = findMenuItem("unfriend");

    if (unfriendOption) {
      unfriendOption.click();
      await randomDelay(250, 450);

      if (checkAndDismissNotFriendsDialog()) {
        console.log(`⏭️ Skipped ${name} (already not friends). Moving on.`);
        return scheduleNext();
      }

      const confirmBtn = findMenuItem("confirm") || findMenuItem("remove");

      if (confirmBtn) {
        confirmBtn.click();
        count++;
        updateStatusBar();
        console.log(`✅ Unfriended #${count} / ${totalFriendsNum || '?'}: ${name}`);
      } else {
        console.log(`⚠️ Confirm dialog not found for ${name}.`);
        document.body.click();
      }
    } else {
      console.log(`⚠️ 'Unfriend' not in menu for ${name} — closing.`);
      document.body.click();
    }

    scheduleNext();
  }

  function scheduleNext() {
    if (running) randomDelay(400, 800).then(unfriendNext);
  }

  setInterval(() => {
    if (checkAndDismissNotFriendsDialog()) {
      console.log("🕵️ Periodic check caught a stuck dialog and cleared it.");
    }
  }, 3 * 60 * 1000);

  totalFriendsNum = getTotalFriendsCount();
  createStatusBar();
  updateStatusBar();
  console.log(`🚀 Starting bulk unfriend. Detected total friends: ${totalFriendsNum || "not found"}. Run stopUnfriend() to halt.`);

  window.stopUnfriend = () => stopEverything("⏹ Stopped manually.");

  unfriendNext();
})();
```

## Known limitations

- Relies on matching visible text and `aria-label` attributes in Facebook's current UI. If Facebook changes its markup (which happens periodically), the selectors in `findFriendCardButtons()` or `findMenuItem()` may need updating.
- Only tested against the desktop web version of Facebook (`facebook.com`), not the mobile site or app.
- Language-dependent: if your Facebook UI is set to a language other than English, the text-matching (e.g., `"unfriend"`, `"confirm"`, `"remove"`) will need to be translated accordingly.

## Contributing

If Facebook changes its layout and this script breaks, PRs updating the selectors are welcome. The most useful debugging step is usually to open DevTools, inspect one friend card, and check what the current `aria-label` or button structure looks like — then adjust `findFriendCardButtons()` accordingly.

## License

MIT — use, modify, and share freely.
