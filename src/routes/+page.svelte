
<script lang="ts">
  import { onMount, tick } from "svelte";
  import { SplashScreen } from "@capacitor/splash-screen";

  type Note = {
    title: string;
    content: string;
    customTitle: boolean;
  };

  const STORAGE_KEY = "quicksave";
  const MAX_NOTE_NAME_LENGTH = 32;
  const fontSizes = [14, 16, 18, 20, 24];

  const emptyNote = (): Note => ({
    title: "",
    content: "",
    customTitle: false
  });

  let notes = $state<Note[]>([emptyNote()]);
  let active = $state(0);

  let font = $state("Funnel Display");
  let fontSize = $state(16);
  let darkMode = $state(false);

  let attrOpen = $state(false);
  let contextOpen = $state(false);
  let contextNote = $state(-1);
  let contextX = $state(0);
  let contextY = $state(0);
  let contextBelow = $state(false);
  let isTyping = $state(false);

  let renameOpen = $state(false);
  let renameValue = $state("");

  let editor: HTMLDivElement;
  let savedRange: Range | null = null;

  let boldActive = $state(false);
  let italicActive = $state(false);
  let underlineActive = $state(false);
  let bulletActive = $state(false);
  let orderedActive = $state(false);

  const plainText = (html: string) => {
    const div = document.createElement("div");
    div.innerHTML = html;
    return (div.innerText || div.textContent || "")
      .replace(/\u00a0/g, " ")
      .replace(/\n{3,}/g, "\n\n")
      .trimEnd();
  };

  function getAutoTitle(content: string) {
    return plainText(content)
      .replace(/\s+/g, " ")
      .trim()
      .slice(0, MAX_NOTE_NAME_LENGTH);
  }

  function getNoteTitle(note: Note) {
    if (note.customTitle) return note.title || "new";
    return getAutoTitle(note.content) || "new";
  }

  function persist() {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(notes));
      localStorage.setItem("quicksave-active", String(active));
    } catch (error) {
      console.error("Could not save notes:", error);
    }
  }

  function updateEditor() {
    if (!editor || !notes[active]) return;

    const html = editor.innerHTML;
    const note = notes[active];

    note.content = html === "<br>" ? "" : html;

    if (!note.customTitle) {
      note.title = getAutoTitle(note.content);
    }

    persist();
  }

  function saveSelection() {
    const selection = window.getSelection();

    if (
      !selection?.rangeCount ||
      !editor?.contains(selection.anchorNode)
    ) return;

    savedRange = selection.getRangeAt(0).cloneRange();
  }

  function restoreSelection() {
    if (!savedRange || !editor) return;

    editor.focus();

    const selection = window.getSelection();
    selection?.removeAllRanges();
    selection?.addRange(savedRange);
  }

  function updateFormattingState() {
    if (!editor) return;

    const selection = window.getSelection();

    if (!selection?.anchorNode || !editor.contains(selection.anchorNode)) {
      return;
    }

    boldActive = document.queryCommandState("bold");
    italicActive = document.queryCommandState("italic");
    underlineActive = document.queryCommandState("underline");
    bulletActive = document.queryCommandState("insertUnorderedList");
    orderedActive = document.queryCommandState("insertOrderedList");

    saveSelection();
  }

  function exec(command: string) {
    restoreSelection();
    document.execCommand(command);
    saveSelection();
    updateFormattingState();
    updateEditor();
  }

  function replaceRange(
  node: Node,
  start: number,
  end: number,
  content: string,
  command: string
) {
  const range = document.createRange();
  range.setStart(node, start);
  range.setEnd(node, end);

  const selection = window.getSelection();
  selection?.removeAllRanges();
  selection?.addRange(range);

  document.execCommand("insertText", false, content);

  const current = window.getSelection();
  if (!current?.rangeCount) return;

  const caret = current.getRangeAt(0);
  const endNode = caret.startContainer;
  const endOffset = caret.startOffset;

  if (endNode.nodeType !== Node.TEXT_NODE) return;

  const formatRange = document.createRange();
  formatRange.setStart(endNode, endOffset - content.length);
  formatRange.setEnd(endNode, endOffset);

  current.removeAllRanges();
  current.addRange(formatRange);

  document.execCommand(command);

  const next = window.getSelection();
  if (!next?.rangeCount) return;

  next.getRangeAt(0).collapse(false);

  saveSelection();
  updateFormattingState();
  updateEditor();
}

  function replaceLink(
    node: Node,
    start: number,
    end: number,
    text: string,
    url: string
  ) {
    let safeUrl: URL;

    try {
      safeUrl = new URL(
        url.startsWith("www.") ? `https://${url}` : url
      );
    } catch {
      return;
    }

    if (!["http:", "https:"].includes(safeUrl.protocol)) return;

    const range = document.createRange();
    range.setStart(node, start);
    range.setEnd(node, end);

    const anchor = document.createElement("a");
    anchor.href = safeUrl.href;
    anchor.textContent = text;
    anchor.target = "_blank";
    anchor.rel = "noopener noreferrer";

    range.deleteContents();
    range.insertNode(anchor);

    const next = document.createRange();
    next.setStartAfter(anchor);
    next.collapse(true);

    const selection = window.getSelection();
    selection?.removeAllRanges();
    selection?.addRange(next);

    saveSelection();
    updateEditor();
  }

  function convertMarkdownFormatting() {
    if (!editor) return;

    const selection = window.getSelection();
    if (!selection?.rangeCount) return;

    const range = selection.getRangeAt(0);
    const node = range.startContainer;

    if (node.nodeType !== Node.TEXT_NODE) return;

    const value = node.textContent ?? "";
    const cursor = range.startOffset;
    const before = value.slice(0, cursor);
    const trimmed = before.trimEnd();
    const spaces = before.length - trimmed.length;
    const effectiveCursor = cursor - spaces;

    const replace = (
      match: RegExpMatchArray,
      command: string
    ) => {
      replaceRange(
        node,
        effectiveCursor - match[0].length,
        cursor,
        match[1],
        command
      );
    };

    // Markdown links must be checked before raw URLs.
    const link = trimmed.match(/\[([^\]]+)\]\((https?:\/\/[^)\s]+)\)$/);

    if (link) {
      replaceLink(
        node,
        effectiveCursor - link[0].length,
        cursor,
        link[1],
        link[2]
      );
      return;
    }

    // Bold
    const bold = trimmed.match(/\*\*([^*]+)\*\*$/);
    if (bold) return replace(bold, "bold");

    const boldUnderscore = trimmed.match(/__([^_]+)__$/);
    if (boldUnderscore) return replace(boldUnderscore, "bold");

    // Italic
    const italic = trimmed.match(/(?<!\*)\*([^*]+)\*(?!\*)$/);
    if (italic) return replace(italic, "italic");

    const italicUnderscore = trimmed.match(/(?<!_)_([^_]+)_(?!_)$/);
    if (italicUnderscore) return replace(italicUnderscore, "italic");

    // Raw URLs
    const url = trimmed.match(/(https?:\/\/[^\s]+)$/);

    if (url) {
      replaceLink(
        node,
        effectiveCursor - url[0].length,
        cursor,
        url[0],
        url[0]
      );
    }
  }

  function handleKeydown(event: KeyboardEvent) {
    const mod = event.ctrlKey || event.metaKey;

    if (mod && ["b", "i", "u"].includes(event.key.toLowerCase())) {
      event.preventDefault();

      const commands: Record<string, string> = {
        b: "bold",
        i: "italic",
        u: "underline"
      };

      exec(commands[event.key.toLowerCase()]);
      return;
    }

    if (event.key === " " && !mod) {
      const selection = window.getSelection();

      if (!selection?.rangeCount) return;

      const range = selection.getRangeAt(0);
      const node = range.startContainer;

      if (node.nodeType !== Node.TEXT_NODE) return;

      const before = (node.textContent ?? "").slice(0, range.startOffset);

      const listMatch = before.match(/^([-*+]|\d+\.) $/);

      if (listMatch) {
        event.preventDefault();

        const ordered = /^\d+\./.test(listMatch[0]);

        document.execCommand(
          ordered ? "insertOrderedList" : "insertUnorderedList"
        );

        const newRange = document.createRange();
        newRange.setStart(node, 0);
        newRange.setEnd(node, range.startOffset);

        newRange.deleteContents();

        saveSelection();
        updateFormattingState();
        updateEditor();
        return;
      }
    }

    if (
      [" ", "Enter", "*", "_", ")"].includes(event.key)
    ) {
      requestAnimationFrame(convertMarkdownFormatting);
    }
  }

  async function loadEditor() {
    await tick();
    if (!editor) return;

    editor.innerHTML = notes[active]?.content ?? "";
    savedRange = null;
    updateFormattingState();
  }

  function newNote() {
    updateEditor();

    notes = [...notes, emptyNote()];
    active = notes.length - 1;

    persist();
    loadEditor();
  }

  function switchNote(index: number) {
    if (index === active) return;

    updateEditor();
    active = index;

    persist();
    closeContextMenu();
    loadEditor();
  }

  function deleteNote(index: number) {
    if (notes.length <= 1) return;

    notes.splice(index, 1);

    if (index < active) {
      active--;
    } else if (index === active) {
      active = Math.min(index, notes.length - 1);
    }

    persist();
    closeContextMenu();
    loadEditor();
  }

  function openContextMenu(
    event: MouseEvent,
    index: number,
    button: HTMLButtonElement
  ) {
    event.preventDefault();
    event.stopPropagation();

    const rect = button.getBoundingClientRect();

    contextNote = index;

    const width = 170;
    const height = 54;

    let x = rect.left + rect.width / 2;
    let y = rect.top - 8;
    let below = false;

    if (y - height < 12) {
      y = rect.bottom + 8;
      below = true;
    }

    x = Math.max(width / 2 + 10, Math.min(
      x,
      window.innerWidth - width / 2 - 10
    ));

    contextX = x;
    contextY = y;
    contextBelow = below;
    contextOpen = true;
    renameOpen = false;
  }

  function closeContextMenu() {
    contextOpen = false;
    contextNote = -1;
  }

  function openRename() {
    if (contextNote < 0) return;

    const note = notes[contextNote];

    renameValue = note.customTitle
      ? note.title
      : getAutoTitle(note.content);

    contextOpen = false;
    renameOpen = true;
  }

  function renameNote() {
    if (contextNote < 0) return;

    const value = renameValue.trim().slice(0, MAX_NOTE_NAME_LENGTH);

    notes[contextNote].title = value;
    notes[contextNote].customTitle = value.length > 0;

    persist();

    renameOpen = false;
    contextNote = -1;
  }

  function cancelRename() {
    renameOpen = false;
    contextNote = -1;
  }

  function safeExportHTML(html: string) {
    const doc = new DOMParser().parseFromString(html, "text/html");

    doc.querySelectorAll(
      "script, iframe, object, embed, form, style, link, meta"
    ).forEach(element => element.remove());

    doc.body.querySelectorAll("*").forEach(element => {
      [...element.attributes].forEach(attribute => {
        const name = attribute.name.toLowerCase();
        const value = attribute.value.trim().toLowerCase();

        if (
          name.startsWith("on") ||
          name === "srcdoc" ||
          (
            ["href", "src", "xlink:href"].includes(name) &&
            value.startsWith("javascript:")
          )
        ) {
          element.removeAttribute(attribute.name);
        }
      });
    });

    return doc.body.innerHTML;
  }

  function downloadNote() {
    if (contextNote < 0) return;

    const note = notes[contextNote];
    const title = getNoteTitle(note) || "Note";
    const safeTitle = title.replace(/[&<>"']/g, char => ({
      "&": "&amp;",
      "<": "&lt;",
      ">": "&gt;",
      '"': "&quot;",
      "'": "&#39;"
    })[char]!);

    const content = safeExportHTML(note.content);
    const printWindow = window.open("", "_blank");

    if (!printWindow) {
      closeContextMenu();
      return;
    }

    printWindow.document.open();
    printWindow.document.write(`
      <!DOCTYPE html>
      <html>
        <head>
          <meta charset="UTF-8">
          <title>${safeTitle}</title>
          <link href="https://fonts.googleapis.com/css2?family=Funnel+Display:wght@400;500;600;700&family=Instrument+Serif&family=Archivo:wght@400;500;600;700&display=swap" rel="stylesheet">
          <style>
            body {
              font-family: "${font}", sans-serif;
              font-size: ${fontSize}px;
              line-height: 1.7;
              color: #20201e;
              max-width: 780px;
              margin: 50px auto;
              padding: 0 30px;
              overflow-wrap: anywhere;
            }
            h1 { font-family: "Instrument Serif", serif; }
            ul, ol { padding-left: 28px; }
            a { color: #496dbe; }
            @media print {
              body { margin: 0; padding: 0; }
            }
          </style>
        </head>
        <body>${content}</body>
      </html>
    `);
    printWindow.document.close();

    const print = () => {
      printWindow.focus();
      printWindow.print();
      printWindow.onafterprint = () => printWindow.close();
    };

    printWindow.onload = print;
    setTimeout(() => {
      if (!printWindow.closed) print();
    }, 700);

    closeContextMenu();
  }

  function changeFont() {
    localStorage.setItem("quicksave-font", font);
  }

  function changeFontSize() {
    const index = fontSizes.indexOf(fontSize);
    fontSize = fontSizes[(index + 1) % fontSizes.length];

    localStorage.setItem("quicksave-font-size", String(fontSize));
  }

  function toggleDarkMode() {
    darkMode = !darkMode;
    localStorage.setItem("quicksave-dark", String(darkMode));
  }

  function load() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);

      if (raw) {
        const saved = JSON.parse(raw);

        if (Array.isArray(saved)) {
          if (
            saved.length === 0 ||
            typeof saved[0] === "string"
          ) {
            notes = saved.map((content: string) => ({
              ...emptyNote(),
              content: typeof content === "string" ? content : ""
            }));
          } else {
            notes = saved.map((note: Partial<Note>) => ({
              title: typeof note.title === "string" ? note.title : "",
              content: typeof note.content === "string" ? note.content : "",
              customTitle: note.customTitle === true
            }));
          }
        }
      }
    } catch (error) {
      console.error("Could not load notes:", error);
    }

    if (!notes.length) notes = [emptyNote()];

    const savedActive = Number(
      localStorage.getItem("quicksave-active")
    );

    if (
      Number.isInteger(savedActive) &&
      savedActive >= 0 &&
      savedActive < notes.length
    ) {
      active = savedActive;
    }

    const savedFont = localStorage.getItem("quicksave-font");
    const savedSize = Number(localStorage.getItem("quicksave-font-size"));
    const savedTheme = localStorage.getItem("quicksave-dark");

    if (
      savedFont &&
      ["Funnel Display", "Archivo", "Instrument Serif"].includes(savedFont)
    ) {
      font = savedFont;
    }

    if (fontSizes.includes(savedSize)) {
      fontSize = savedSize;
    }

    if (savedTheme !== null) {
      darkMode = savedTheme === "true";
    }
  }

  function closeOnEscape(event: KeyboardEvent) {
    if (event.key === "Escape") {
      closeContextMenu();
      cancelRename();
      attrOpen = false;
    }
  }

  onMount(() => {
    load();
    loadEditor();

    SplashScreen.hide().catch(() => {});

    const close = (event: MouseEvent) => {
      const target = event.target as HTMLElement;

      if (!target.closest(".context-menu")) {
        closeContextMenu();
      }
    };

    document.addEventListener("click", close);
    document.addEventListener("keydown", closeOnEscape);
    document.addEventListener("selectionchange", updateFormattingState);

    return () => {
      document.removeEventListener("click", close);
      document.removeEventListener("keydown", closeOnEscape);
      document.removeEventListener("selectionchange", updateFormattingState);
    };
  });
</script>

<svelte:head>
  <title>Quicksave — a little place to think</title>
  <meta name="theme-color" content={darkMode ? "#191a1c" : "#f7f7f4"} />
  <meta name="description" content="A tiny, local-first space for your thoughts." />

  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link href="https://fonts.googleapis.com/css2?family=Funnel+Display:wght@400;500;600;700&family=Archivo:wght@400;450;500;550;600;650;700&family=Instrument+Serif:ital@0;1&display=swap" rel="stylesheet" />
</svelte:head>

<main class:dark={darkMode} class="app">
  <header class="topbar">
    <a class="wordmark" href="/" aria-label="Quicksave home">
      quicksave<span>.</span>
    </a>

    <div class="top-right">

      <button
        class="icon-button about-button"
        onclick={() => attrOpen = true}
        aria-label="About Quicksave"
        title="About"
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <circle cx="12" cy="12" r="9" />
          <path d="M12 11v5m0-8h.01" stroke-linecap="round" />
        </svg>
      </button>
    </div>
  </header>

  <section class="workspace">

    <div
      bind:this={editor}
      class="editor"
      contenteditable="true"
      onfocus={() => (isTyping = true)}
      onblur={() => (isTyping = false)}
      role="textbox"
      aria-label="Note editor"
      aria-multiline="true"
      data-placeholder="Start with a thought..."
      spellcheck="true"
      oninput={updateEditor}
      onkeydown={handleKeydown}
      onclick={updateFormattingState}
      onkeyup={updateFormattingState}
      onmouseup={updateFormattingState}
      style={`font-family: "${font}", sans-serif; font-size: ${fontSize}px;`}
    ></div>

    <div class="editor-footer" class:hidden={isTyping}>
      <span>⌘ / Ctrl + B · I · U</span>
    </div>
  </section>

  <footer class="toolbar">
    <div class="note-strip">
      {#each notes as note, i}
        <button
          class="note-tab"
          class:active={i === active}
          onclick={() => switchNote(i)}
          oncontextmenu={(event) =>
            openContextMenu(event, i, event.currentTarget as HTMLButtonElement)}
          title={getNoteTitle(note)}
        >
          <span class="tab-indicator"></span>
          <span class="tab-name">{getNoteTitle(note)}</span>
        </button>
      {/each}

      <button class="new-note" onclick={newNote} title="New note" aria-label="New note">
        +
      </button>
    </div>

    <div class="toolbar-divider"></div>

    <div class="format-controls">
      <button class="tool-button" class:pressed={boldActive} onclick={() => exec("bold")} title="Bold (Ctrl+B)">
        <strong>B</strong>
      </button>

      <button class="tool-button" class:pressed={italicActive} onclick={() => exec("italic")} title="Italic (Ctrl+I)">
        <em>I</em>
      </button>

      <button class="tool-button" class:pressed={underlineActive} onclick={() => exec("underline")} title="Underline (Ctrl+U)">
        <u>U</u>
      </button>

      <span class="mini-divider"></span>

      <button class="tool-button list-button" class:pressed={bulletActive} onclick={() => exec("insertUnorderedList")} title="Bullet list">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <path d="M9 6h11M9 12h11M9 18h11" stroke-linecap="round" />
          <circle cx="4" cy="6" r="1" fill="currentColor" />
          <circle cx="4" cy="12" r="1" fill="currentColor" />
          <circle cx="4" cy="18" r="1" fill="currentColor" />
        </svg>
      </button>

      <button class="tool-button list-button" class:pressed={orderedActive} onclick={() => exec("insertOrderedList")} title="Numbered list">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <path d="M10 6h10M10 12h10M10 18h10" stroke-linecap="round" />
          <path d="M3 5h2v3M3 12h2m-2 1h2M3 17h2v2H3" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>
    </div>

    <div class="toolbar-divider secondary-divider"></div>

    <div class="settings-controls">
      <select
        class="font-select"
        bind:value={font}
        onchange={changeFont}
        aria-label="Editor font"
        title="Font"
      >
        <option value="Funnel Display">Funnel</option>
        <option value="Archivo">Archivo</option>
        <option value="Instrument Serif">Instrument</option>
      </select>

      <button class="tool-button size-button" onclick={changeFontSize} title="Change font size">
        {fontSize}
      </button>

      <button class="tool-button theme-button" onclick={toggleDarkMode} title="Toggle theme">
        {#if darkMode}
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
            <circle cx="12" cy="12" r="4" />
            <path d="M12 2v2m0 16v2M4.93 4.93l1.42 1.42m11.3 11.3 1.42 1.42M2 12h2m16 0h2M4.93 19.07l1.42-1.42m11.3-11.3 1.42-1.42" stroke-linecap="round" />
          </svg>
        {:else}
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
            <path d="M20.5 14.2A8.5 8.5 0 0 1 9.8 3.5 8.5 8.5 0 1 0 20.5 14.2Z" stroke-linejoin="round" />
          </svg>
        {/if}
      </button>
    </div>
  </footer>

  {#if contextOpen}
    <div
      class="context-menu"
      style={`left:${contextX}px;top:${contextY}px;transform:translate(-50%, ${contextBelow ? "0" : "-100%"})`}
      onclick={(event) => event.stopPropagation()}
      oncontextmenu={(event) => event.preventDefault()}
    >
      <button onclick={openRename} title="Rename note">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <path d="m14 5 5 5M4 20l4-.8L19 8a2.12 2.12 0 0 0-3-3L5 16l-1 4Z" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>

      <button onclick={downloadNote} title="Save as PDF">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <path d="M12 3v12m-5-5 5 5 5-5M4 17v3h16v-3" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>

      <button
        onclick={() => deleteNote(contextNote)}
        title="Delete note"
        disabled={notes.length <= 1}
      >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7">
          <path d="M4 7h16M10 11v6m4-6v6M5 7l1 14h12l1-14M9 7V4h6v3" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>
    </div>
  {/if}

  {#if renameOpen}
    <div class="overlay" onclick={cancelRename}>
      <section class="dialog" onclick={(event) => event.stopPropagation()}>
        <span class="eyebrow">NOTE SETTINGS</span>
        <h2>Give it a name.</h2>
        <p>A little label to find it again.</p>

        <input
          class="rename-input"
          bind:value={renameValue}
          maxlength={MAX_NOTE_NAME_LENGTH}
          autofocus
          placeholder="Untitled"
          onkeydown={(event) => {
            if (event.key === "Enter") renameNote();
          }}
        />

        <div class="dialog-footer">
          <span>{renameValue.length}/{MAX_NOTE_NAME_LENGTH}</span>
          <div>
            <button class="dialog-button" onclick={cancelRename}>Cancel</button>
            <button class="dialog-button primary" onclick={renameNote}>Save name</button>
          </div>
        </div>
      </section>
    </div>
  {/if}

  {#if attrOpen}
    <div class="overlay" onclick={() => attrOpen = false}>
      <section class="dialog about-dialog" onclick={(event) => event.stopPropagation()}>
        <span class="eyebrow">A SMALL THING, MADE WITH CARE</span>
        <h2>Quicksave<span>.</span></h2>
        <p>
          A tiny, local-first space for thoughts, ideas, and things worth keeping.
          No accounts. No feeds. Just you and your notes.
        </p>

        <div class="about-bottom">
          <a href="https://maniksharma.xyz/" target="_blank" rel="noopener noreferrer">
            Made by Manik Sharma ↗
          </a>
          <button class="dialog-button" onclick={() => attrOpen = false}>Close</button>
        </div>
      </section>
    </div>
  {/if}
</main>

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(html) {
    background: #f7f7f4;
    color-scheme: light;
  }

  :global(body) {
    margin: 0;
    min-width: 320px;
    min-height: 100vh;
    font-family: "Archivo", sans-serif;
  }

  :global(button), :global(input), :global(select) {
    font: inherit;
  }

  :global(button:focus-visible),
  :global(input:focus-visible),
  :global(select:focus-visible),
  :global(a:focus-visible) {
    outline: 2px solid #668d73;
    outline-offset: 3px;
  }

  .app {
    --bg: #f7f7f4;
    --surface: #ffffff;
    --surface-hover: #efefea;
    --text: #292b27;
    --muted: #93958e;
    --subtle: #b1b2aa;
    --line: #e8e8e1;
    --accent: #587660;
    --accent-soft: #e9efe9;
    --shadow: 0 12px 40px rgba(36, 39, 32, .055);

    display: flex;
    flex-direction: column;
    width: 100%;
    min-height: 100vh;
    height: 100dvh;
    overflow: hidden;
    background: var(--bg);
    color: var(--text);
    transition: background .2s, color .2s;
  }

  .app.dark {
    --bg: #191a1c;
    --surface: #222326;
    --surface-hover: #2d2e31;
    --text: #edede8;
    --muted: #92938e;
    --subtle: #73756f;
    --line: #333438;
    --accent: #a4bea6;
    --accent-soft: #2b372e;
    --shadow: 0 12px 40px rgba(0, 0, 0, .18);

    color-scheme: dark;
  }

  .topbar {
    height: 74px;
    min-height: 74px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 clamp(20px, 5vw, 76px);
    border-bottom: 1px solid var(--line);
  }

  .wordmark {
    font-family: "Funnel Display", sans-serif;
    font-weight: 650;
    font-size: 23px;
    letter-spacing: -1.2px;
    color: var(--text);
    text-decoration: none;
  }

  .wordmark span, .about-dialog h2 span {
    color: var(--accent);
  }

  .top-right {
    display: flex;
    align-items: center;
    gap: 23px;
  }

  .save-status {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 11px;
    letter-spacing: .02em;
    color: var(--muted);
  }

  .status-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: #80a78a;
    box-shadow: 0 0 0 3px var(--accent-soft);
  }

  .icon-button, .tool-button, .new-note {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: 0;
    cursor: pointer;
    color: var(--muted);
    background: transparent;
    transition: background .15s, color .15s;
  }

  .icon-button svg {
    width: 19px;
    height: 19px;
  }

  .icon-button {
    width: 34px;
    height: 34px;
    border-radius: 50%;
  }

  .icon-button:hover, .tool-button:hover, .new-note:hover {
    color: var(--text);
    background: var(--surface-hover);
  }

  .workspace {
    flex: 1;
    min-height: 0;
    display: flex;
    flex-direction: column;
    width: min(100%, 1000px);
    margin: 0 auto;
    padding: 38px clamp(22px, 7vw, 90px) 18px;
  }

  .editor-meta {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding-bottom: 24px;
  }

  .eyebrow {
    color: var(--muted);
    font-family: "Archivo", sans-serif;
    font-size: 10px;
    font-weight: 650;
    letter-spacing: .14em;
  }

  .note-count {
    color: var(--subtle);
    font-size: 11px;
  }

  .editor {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    outline: none;
    color: var(--text);
    line-height: 1.8;
    overflow-wrap: anywhere;
    caret-color: var(--accent);
    scrollbar-width: thin;
    scrollbar-color: var(--line) transparent;
  }

  .editor:empty::before {
    content: attr(data-placeholder);
    color: var(--subtle);
    pointer-events: none;
  }

  .editor strong, .editor b {
    font-weight: 700;
  }

  .editor em, .editor i {
    font-style: italic;
  }

  .editor u {
    text-decoration-thickness: 1px;
    text-underline-offset: 3px;
  }

  .editor ul, .editor ol {
    padding-left: 27px;
    margin: 8px 0;
  }

  .editor li {
    padding-left: 4px;
    margin: 4px 0;
  }

  .editor a {
    color: var(--accent);
    text-decoration-thickness: 1px;
    text-underline-offset: 3px;
  }

  .editor blockquote {
    margin: 12px 0;
    padding-left: 17px;
    border-left: 2px solid var(--accent);
    color: var(--muted);
  }

  .editor-footer {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    padding-top: 19px;
    color: var(--subtle);
    font-size: 10px;
  }

  .toolbar {
    position: relative;
    display: flex;
    align-items: center;
    gap: 13px;
    min-height: 76px;
    padding: 10px clamp(12px, 3vw, 40px);
    border-top: 1px solid var(--line);
    background: var(--surface);
  }

  .note-strip {
    display: flex;
    align-items: center;
    gap: 5px;
    min-width: 0;
    flex: 1;
    overflow-x: auto;
    scrollbar-width: none;
  }

  .note-strip::-webkit-scrollbar {
    display: none;
  }

  .note-tab {
    position: relative;
    display: flex;
    align-items: center;
    gap: 8px;
    height: 38px;
    max-width: 180px;
    flex-shrink: 0;
    padding: 0 12px;
    border: 1px solid transparent;
    border-radius: 7px;
    background: transparent;
    color: var(--muted);
    cursor: pointer;
    font-family: "Archivo", sans-serif;
    font-size: 12px;
    transition: background .15s, border .15s, color .15s;
  }

  .note-tab:hover {
    background: var(--surface-hover);
    color: var(--text);
  }

  .note-tab.active {
    color: var(--text);
    background: var(--bg);
    border-color: var(--line);
  }

  .tab-indicator {
    width: 5px;
    height: 5px;
    border-radius: 50%;
    background: transparent;
    flex-shrink: 0;
  }

  .note-tab.active .tab-indicator {
    background: var(--accent);
  }

  .tab-name {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .new-note {
    width: 32px;
    height: 32px;
    flex-shrink: 0;
    border: 1px dashed var(--line);
    border-radius: 7px;
    font-size: 21px;
    font-weight: 300;
  }

  .toolbar-divider {
    width: 1px;
    height: 29px;
    background: var(--line);
    flex-shrink: 0;
  }

  .format-controls, .settings-controls {
    display: flex;
    align-items: center;
    gap: 3px;
    flex-shrink: 0;
  }

  .tool-button {
    width: 34px;
    height: 34px;
    border-radius: 6px;
    font-family: "Archivo", sans-serif;
    font-size: 13px;
  }

  .tool-button.pressed {
    background: var(--accent-soft);
    color: var(--accent);
  }

  .tool-button svg {
    width: 17px;
    height: 17px;
  }

  .mini-divider {
    width: 1px;
    height: 19px;
    margin: 0 4px;
    background: var(--line);
  }

  .font-select {
    max-width: 112px;
    height: 34px;
    padding: 0 7px;
    border: 0;
    border-radius: 6px;
    outline: none;
    background: transparent;
    color: var(--muted);
    font-family: "Archivo", sans-serif;
    font-size: 11px;
    cursor: pointer;
  }

  .font-select:hover {
    background: var(--surface-hover);
    color: var(--text);
  }

  .size-button {
    width: 37px;
    font-size: 11px;
  }

  .theme-button svg {
    width: 17px;
    height: 17px;
  }

  .context-menu {
    position: fixed;
    z-index: 2000;
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 6px;
    border: 1px solid var(--line);
    border-radius: 11px;
    background: var(--surface);
    box-shadow: var(--shadow);
  }

  .context-menu button {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 39px;
    height: 37px;
    border: 0;
    border-radius: 7px;
    background: transparent;
    color: var(--muted);
    cursor: pointer;
  }

  .context-menu button:hover:not(:disabled) {
    color: var(--text);
    background: var(--surface-hover);
  }

  .context-menu button:disabled {
    opacity: .25;
    cursor: not-allowed;
  }

  .context-menu svg {
    width: 17px;
    height: 17px;
  }

  .overlay {
    position: fixed;
    inset: 0;
    z-index: 3000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 20px;
    background: rgba(15, 16, 14, .35);
    backdrop-filter: blur(5px);
  }

  .dialog {
    width: min(420px, 100%);
    padding: 30px;
    border: 1px solid var(--line);
    border-radius: 13px;
    background: var(--surface);
    color: var(--text);
    box-shadow: 0 20px 70px rgba(0, 0, 0, .12);
    animation: appear .16s ease-out;
  }

  @keyframes appear {
    from { opacity: 0; transform: translateY(6px); }
    to { opacity: 1; transform: translateY(0); }
  }

  .dialog h2 {
    margin: 12px 0 5px;
    font-family: "Instrument Serif", serif;
    font-size: 35px;
    font-weight: 400;
    letter-spacing: -.5px;
  }

  .dialog p {
    margin: 0 0 22px;
    color: var(--muted);
    font-size: 13px;
    line-height: 1.7;
  }

  .rename-input {
    width: 100%;
    height: 43px;
    padding: 0 12px;
    border: 1px solid var(--line);
    border-radius: 7px;
    outline: none;
    background: var(--bg);
    color: var(--text);
    font-size: 13px;
  }

  .rename-input:focus {
    border-color: var(--accent);
  }

  .dialog-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 17px;
  }

  .dialog-footer > span {
    color: var(--subtle);
    font-size: 10px;
  }

  .dialog-footer > div {
    display: flex;
    gap: 7px;
  }

  .dialog-button {
    height: 34px;
    padding: 0 12px;
    border: 1px solid var(--line);
    border-radius: 6px;
    background: transparent;
    color: var(--muted);
    cursor: pointer;
    font-size: 11px;
  }

  .dialog-button:hover {
    color: var(--text);
    background: var(--surface-hover);
  }

  .dialog-button.primary {
    border-color: var(--accent);
    background: var(--accent);
    color: white;
  }

  .about-dialog {
    padding: 34px;
  }

  .about-dialog h2 {
    font-size: 43px;
    letter-spacing: -1.5px;
  }

  .about-dialog p {
    max-width: 310px;
  }

  .about-bottom {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    padding-top: 15px;
    border-top: 1px solid var(--line);
  }

  .about-bottom a {
    color: var(--accent);
    font-size: 11px;
    text-decoration: none;
  }

  .about-bottom a:hover {
    text-decoration: underline;
  }

  @media (max-width: 760px) {
    .topbar {
      height: 63px;
      min-height: 63px;
    }

    .workspace {
      padding-top: 28px;
    }

    .toolbar {
      flex-wrap: wrap;
      gap: 8px;
      padding: 10px 12px;
    }

    .note-strip {
      flex-basis: 100%;
      min-height: 35px;
    }

    .toolbar-divider:first-of-type {
      display: none;
    }

    .format-controls {
      margin-right: auto;
    }

    .secondary-divider {
      display: none;
    }

    .editor-footer {
      font-size: 9px;
    }
  }

  @media (max-width: 390px) {
    .font-select {
      max-width: 84px;
    }

    .toolbar {
      gap: 5px;
    }

    .tool-button {
      width: 31px;
    }

    .workspace {
      padding-left: 19px;
      padding-right: 19px;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      scroll-behavior: auto !important;
      animation-duration: .01ms !important;
      transition-duration: .01ms !important;
    }
  }
</style>
