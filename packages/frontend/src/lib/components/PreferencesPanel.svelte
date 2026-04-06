<script lang="ts">
  import { onMount } from 'svelte';

  interface Preferences {
    identity: { companion_name: string; user_name: string; timezone: string };
    agent: { model: string; model_autonomous: string };
    orchestrator: { enabled: boolean };
    voice: { enabled: boolean };
    discord: { enabled: boolean };
    telegram: { enabled: boolean };
    auth: { has_password: boolean };
  }

  let prefs = $state<Preferences | null>(null);
  let loading = $state(true);
  let saving = $state(false);
  let message = $state<string | null>(null);
  let error = $state<string | null>(null);

  // Editable drafts
  let companionName = $state('');
  let userName = $state('');
  let timezone = $state('');
  let model = $state('');
  let modelAutonomous = $state('');
  let orchestratorEnabled = $state(true);
  let voiceEnabled = $state(false);
  let discordEnabled = $state(false);
  let telegramEnabled = $state(false);
  let newPassword = $state('');

  // Theme + Accent
  const THEMES = [
    { id: 'rose',  label: 'Midnight', bg: '#111113', accent: '#909090' },
    { id: 'petal', label: 'Daylight', bg: '#f0f0ee', accent: '#505050' },
  ];

  let currentTheme = $state(
    typeof localStorage !== 'undefined'
      ? (localStorage.getItem('resonant-theme') ?? 'rose')
      : 'rose'
  );

  function setTheme(id: string) {
    currentTheme = id;
    document.documentElement.setAttribute('data-theme', id);
    localStorage.setItem('resonant-theme', id);
  }

  const ACCENTS = [
    { id: 'crimson',  label: 'Crimson',  color: '#c43040' },
    { id: 'burgundy', label: 'Burgundy', color: '#d01850' },
    { id: 'rose',     label: 'Rose',     color: '#c04068' },
    { id: 'orange',   label: 'Orange',   color: '#d87818' },
    { id: 'amber',    label: 'Amber',    color: '#c88818' },
    { id: 'forest',   label: 'Forest',   color: '#1e7840' },
    { id: 'emerald',  label: 'Emerald',  color: '#1a9868' },
    { id: 'mint',     label: 'Mint',     color: '#1aaa90' },
    { id: 'teal',     label: 'Teal',     color: '#18b8a8' },
    { id: 'ocean',    label: 'Ocean',    color: '#1880c0' },
    { id: 'sapphire', label: 'Sapphire', color: '#1848c8' },
    { id: 'lavender', label: 'Lavender', color: '#8068d0' },
    { id: 'amethyst', label: 'Amethyst', color: '#6040b8' },
    { id: 'plum',     label: 'Plum',     color: '#7020a0' },
    { id: 'magenta',  label: 'Magenta',  color: '#c81878' },
    { id: 'blush',    label: 'Blush',    color: '#b88890' },
    { id: 'slate',    label: 'Slate',    color: '#707070' },
    { id: 'silver',   label: 'Silver',   color: '#a0a0a0' },
  ];

  let currentAccent = $state(
    typeof localStorage !== 'undefined'
      ? (localStorage.getItem('resonant-accent') ?? '')
      : ''
  );

  function setAccent(id: string) {
    currentAccent = id;
    if (id) {
      document.documentElement.setAttribute('data-accent', id);
      localStorage.setItem('resonant-accent', id);
    } else {
      document.documentElement.removeAttribute('data-accent');
      localStorage.removeItem('resonant-accent');
    }
  }

  const MODELS = [
    { id: 'claude-opus-4-6', label: 'Claude Opus 4.6' },
    { id: 'claude-opus-4-5', label: 'Claude Opus 4.5' },
    { id: 'claude-sonnet-4-6', label: 'Claude Sonnet 4.6' },
    { id: 'claude-haiku-4-5', label: 'Claude Haiku 4.5' },
  ];

  const COMMON_TIMEZONES = [
    'UTC',
    'America/New_York', 'America/Chicago', 'America/Denver', 'America/Los_Angeles',
    'Europe/London', 'Europe/Paris', 'Europe/Berlin',
    'Asia/Tokyo', 'Asia/Shanghai', 'Asia/Kolkata',
    'Australia/Sydney', 'Pacific/Auckland',
  ];

  async function loadPrefs() {
    try {
      const res = await fetch('/api/preferences');
      if (!res.ok) throw new Error('Failed to load');
      prefs = await res.json();
      // Populate drafts
      companionName = prefs!.identity.companion_name;
      userName = prefs!.identity.user_name;
      timezone = prefs!.identity.timezone;
      model = prefs!.agent.model;
      modelAutonomous = prefs!.agent.model_autonomous;
      orchestratorEnabled = prefs!.orchestrator.enabled;
      voiceEnabled = prefs!.voice.enabled;
      discordEnabled = prefs!.discord.enabled;
      telegramEnabled = prefs!.telegram.enabled;
    } catch (e) {
      error = 'Failed to load preferences';
    } finally {
      loading = false;
    }
  }

  async function savePrefs() {
    saving = true;
    message = null;
    error = null;
    try {
      const updates: Record<string, unknown> = {
        identity: { companion_name: companionName, user_name: userName, timezone },
        agent: { model, model_autonomous: modelAutonomous },
        orchestrator: { enabled: orchestratorEnabled },
        voice: { enabled: voiceEnabled },
        discord: { enabled: discordEnabled },
        telegram: { enabled: telegramEnabled },
      };
      if (newPassword) {
        updates.auth = { password: newPassword };
      }
      const res = await fetch('/api/preferences', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates),
      });
      const data = await res.json();
      if (res.ok) {
        message = data.message || 'Saved';
        newPassword = '';
      } else {
        error = data.error || 'Failed to save';
      }
    } catch {
      error = 'Failed to save preferences';
    } finally {
      saving = false;
    }
  }

  onMount(loadPrefs);
</script>

<div class="prefs-panel">
  {#if loading}
    <p class="loading-text">Loading preferences...</p>
  {:else if prefs}
    <!-- Appearance -->
    <section class="section">
      <h3 class="section-title">Appearance</h3>
      <p class="section-desc">Pick a base and an accent. Both apply instantly.</p>

      <p class="subsection-label">Base</p>
      <div class="theme-grid">
        {#each THEMES as theme}
          <button
            class="theme-swatch"
            class:active={currentTheme === theme.id}
            onclick={() => setTheme(theme.id)}
            title={theme.label}
            aria-label={theme.label}
            aria-pressed={currentTheme === theme.id}
          >
            <span class="swatch-preview" style="background: {theme.bg};">
              <span class="swatch-dot" style="background: {currentAccent ? (ACCENTS.find(a => a.id === currentAccent)?.color ?? theme.accent) : theme.accent};"></span>
            </span>
            <span class="swatch-label">{theme.label}</span>
          </button>
        {/each}
      </div>

      <p class="subsection-label" style="margin-top: 1.25rem;">Accent</p>
      <div class="accent-grid">
        {#each ACCENTS as accent}
          <button
            class="accent-dot"
            class:active={currentAccent === accent.id}
            onclick={() => setAccent(accent.id)}
            title={accent.label}
            aria-label={accent.label}
            aria-pressed={currentAccent === accent.id}
          >
            <span class="dot-preview" style="background: {accent.color};"></span>
            <span class="dot-label">{accent.label}</span>
          </button>
        {/each}
      </div>
    </section>

    <!-- Identity -->
    <section class="section">
      <h3 class="section-title">Identity</h3>
      <p class="section-desc">Names and timezone used throughout the system.</p>

      <div class="field">
        <label class="field-label" for="pref-companion">Companion Name</label>
        <input id="pref-companion" type="text" class="field-input" bind:value={companionName} placeholder="Echo" />
      </div>

      <div class="field">
        <label class="field-label" for="pref-user">Your Name</label>
        <input id="pref-user" type="text" class="field-input" bind:value={userName} placeholder="Alex" />
      </div>

      <div class="field">
        <label class="field-label" for="pref-tz">Timezone</label>
        <select id="pref-tz" class="field-select" bind:value={timezone}>
          {#each COMMON_TIMEZONES as tz}
            <option value={tz}>{tz}</option>
          {/each}
          {#if !COMMON_TIMEZONES.includes(timezone)}
            <option value={timezone}>{timezone}</option>
          {/if}
        </select>
      </div>
    </section>

    <!-- Agent Models -->
    <section class="section">
      <h3 class="section-title">Agent Models</h3>
      <p class="section-desc">Claude model for interactive and autonomous messages.</p>

      <div class="field">
        <label class="field-label" for="pref-model">Interactive Model</label>
        <select id="pref-model" class="field-select" bind:value={model}>
          {#each MODELS as m}
            <option value={m.id}>{m.label}</option>
          {/each}
        </select>
        <span class="field-hint">Used when you send a message</span>
      </div>

      <div class="field">
        <label class="field-label" for="pref-model-auto">Autonomous Model</label>
        <select id="pref-model-auto" class="field-select" bind:value={modelAutonomous}>
          {#each MODELS as m}
            <option value={m.id}>{m.label}</option>
          {/each}
        </select>
        <span class="field-hint">Used for scheduled wakes and autonomous actions</span>
      </div>
    </section>

    <!-- Toggles -->
    <section class="section">
      <h3 class="section-title">Features</h3>
      <p class="section-desc">Enable or disable system features.</p>

      <label class="toggle-row">
        <input type="checkbox" bind:checked={orchestratorEnabled} />
        <span class="toggle-label">Orchestrator</span>
        <span class="toggle-desc">Scheduled wake-ups and autonomous actions</span>
      </label>

      <label class="toggle-row">
        <input type="checkbox" bind:checked={voiceEnabled} />
        <span class="toggle-label">Voice</span>
        <span class="toggle-desc">ElevenLabs TTS and Groq transcription</span>
      </label>
      {#if voiceEnabled}
        <div class="setup-guide">
          <p class="guide-title">Voice Setup</p>
          <ol class="guide-steps">
            <li>Get an API key from <strong>ElevenLabs</strong> — <a href="https://elevenlabs.io" target="_blank" rel="noopener">elevenlabs.io</a> → Profile → API Keys</li>
            <li>Create or choose a voice, copy the <strong>Voice ID</strong> from the voice settings</li>
            <li>For transcription, get a <strong>Groq</strong> API key — <a href="https://console.groq.com" target="_blank" rel="noopener">console.groq.com</a> → API Keys</li>
            <li>Add to your <code>.env</code> file:
              <pre class="guide-code">ELEVENLABS_API_KEY=your_key_here
ELEVENLABS_VOICE_ID=your_voice_id
GROQ_API_KEY=your_groq_key</pre>
            </li>
            <li>Restart the server</li>
          </ol>
        </div>
      {/if}

      <label class="toggle-row">
        <input type="checkbox" bind:checked={discordEnabled} />
        <span class="toggle-label">Discord</span>
        <span class="toggle-desc">Discord bot gateway integration</span>
      </label>
      {#if discordEnabled}
        <div class="setup-guide">
          <p class="guide-title">Discord Setup</p>
          <ol class="guide-steps">
            <li>Go to the <a href="https://discord.com/developers/applications" target="_blank" rel="noopener">Discord Developer Portal</a></li>
            <li>Create a <strong>New Application</strong>, then go to <strong>Bot</strong> → Reset Token → copy the token</li>
            <li>Under <strong>Privileged Gateway Intents</strong>, enable: Message Content, Server Members, Presence</li>
            <li>Go to <strong>OAuth2</strong> → URL Generator → select <code>bot</code> scope with permissions: Send Messages, Read Message History, Add Reactions, Embed Links, Attach Files</li>
            <li>Use the generated URL to invite the bot to your server</li>
            <li>Right-click your username in Discord → Copy User ID (enable Developer Mode in Discord settings first)</li>
            <li>Add to your <code>.env</code> file:
              <pre class="guide-code">DISCORD_BOT_TOKEN=your_bot_token</pre>
            </li>
            <li>Set your owner user ID in <code>resonant.yaml</code>:
              <pre class="guide-code">discord:
  enabled: true
  owner_user_id: "your_discord_user_id"</pre>
            </li>
            <li>Restart the server. Configure rules in the Discord tab in settings.</li>
          </ol>
        </div>
      {/if}

      <label class="toggle-row">
        <input type="checkbox" bind:checked={telegramEnabled} />
        <span class="toggle-label">Telegram</span>
        <span class="toggle-desc">Telegram bot integration</span>
      </label>
      {#if telegramEnabled}
        <div class="setup-guide">
          <p class="guide-title">Telegram Setup</p>
          <ol class="guide-steps">
            <li>Open Telegram, search for <strong>@BotFather</strong></li>
            <li>Send <code>/newbot</code>, follow the prompts to name your bot</li>
            <li>Copy the <strong>bot token</strong> BotFather gives you</li>
            <li>Send a message to your new bot, then visit:<br/>
              <code>https://api.telegram.org/bot&lt;TOKEN&gt;/getUpdates</code><br/>
              Find your <strong>chat ID</strong> in the response JSON under <code>message.chat.id</code></li>
            <li>Add to your <code>.env</code> file:
              <pre class="guide-code">TELEGRAM_BOT_TOKEN=your_bot_token</pre>
            </li>
            <li>Set your chat ID in <code>resonant.yaml</code>:
              <pre class="guide-code">telegram:
  enabled: true
  owner_chat_id: "your_chat_id"</pre>
            </li>
            <li>Restart the server</li>
          </ol>
        </div>
      {/if}
    </section>

    <!-- Security -->
    <section class="section">
      <h3 class="section-title">Security</h3>
      <p class="section-desc">
        {#if prefs.auth.has_password}
          Password is set. Leave blank to keep current password.
        {:else}
          No password set. Access is open to anyone on the network.
        {/if}
      </p>

      <div class="field">
        <label class="field-label" for="pref-password">
          {prefs.auth.has_password ? 'Change Password' : 'Set Password'}
        </label>
        <input id="pref-password" type="password" class="field-input" bind:value={newPassword} placeholder="Leave blank to keep unchanged" />
      </div>
    </section>

    <!-- Save -->
    <div class="save-area">
      {#if message}
        <p class="save-message success">{message}</p>
      {/if}
      {#if error}
        <p class="save-message error">{error}</p>
      {/if}
      <button class="save-btn" onclick={savePrefs} disabled={saving}>
        {saving ? 'Saving...' : 'Save Preferences'}
      </button>
      <p class="save-hint">Some changes require a server restart to take effect.</p>
    </div>
  {:else}
    <p class="loading-text">{error || 'Unable to load preferences'}</p>
  {/if}
</div>

<style>
  .prefs-panel {
    max-width: 540px;
  }

  .theme-grid {
    display: flex;
    gap: 0.75rem;
    flex-wrap: wrap;
  }

  .theme-swatch {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.375rem;
    padding: 0;
    background: none;
    border: none;
    cursor: pointer;
  }

  .swatch-preview {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 3.5rem;
    height: 2.5rem;
    border-radius: 6px;
    border: 2px solid var(--border);
    transition: border-color var(--transition), box-shadow var(--transition);
  }

  .theme-swatch:hover .swatch-preview {
    border-color: var(--border-hover);
  }

  .theme-swatch.active .swatch-preview {
    border-color: var(--gold);
    box-shadow: 0 0 0 2px var(--gold-glow);
  }

  .swatch-dot {
    width: 0.875rem;
    height: 0.875rem;
    border-radius: 50%;
    opacity: 0.9;
  }

  .swatch-label {
    font-size: 0.75rem;
    color: var(--text-muted);
    letter-spacing: 0.02em;
    transition: color var(--transition-fast);
  }

  .theme-swatch.active .swatch-label {
    color: var(--text-secondary);
  }

  .subsection-label {
    font-size: 0.75rem;
    color: var(--text-muted);
    letter-spacing: 0.06em;
    text-transform: uppercase;
    margin: 0 0 0.625rem;
  }

  .accent-grid {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .accent-dot {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.3rem;
    padding: 0;
    background: none;
    border: none;
    cursor: pointer;
  }

  .dot-preview {
    width: 2.125rem;
    height: 2.125rem;
    border-radius: 50%;
    border: 2.5px solid transparent;
    transition: border-color var(--transition), box-shadow var(--transition);
    flex-shrink: 0;
  }

  .dot-auto {
    background: conic-gradient(
      #c43040 0deg 45deg,
      #d87818 45deg 90deg,
      #1e7840 90deg 135deg,
      #1880c0 135deg 180deg,
      #7020a0 180deg 225deg,
      #c81878 225deg 270deg,
      #c88818 270deg 315deg,
      #1aaa90 315deg 360deg
    );
  }

  .accent-dot:hover .dot-preview {
    border-color: var(--border-hover);
  }

  .accent-dot.active .dot-preview {
    border-color: var(--text-primary);
    box-shadow: 0 0 0 2px var(--gold-glow);
  }

  .dot-label {
    font-size: 0.6875rem;
    color: var(--text-muted);
    letter-spacing: 0.02em;
    transition: color var(--transition-fast);
    white-space: nowrap;
  }

  .accent-dot.active .dot-label {
    color: var(--text-secondary);
  }

  .loading-text {
    color: var(--text-muted);
    font-size: 0.875rem;
    font-style: italic;
    padding: 1rem 0;
  }

  .section {
    margin-bottom: 2rem;
    padding-bottom: 1.5rem;
    border-bottom: 1px solid var(--border);
  }

  .section:last-of-type {
    border-bottom: none;
  }

  .section-title {
    font-family: var(--font-heading);
    font-size: 0.9375rem;
    font-weight: 400;
    color: var(--text-accent);
    letter-spacing: 0.04em;
    margin: 0 0 0.375rem;
  }

  .section-desc {
    font-size: 0.8125rem;
    color: var(--text-muted);
    margin: 0 0 1rem;
    line-height: 1.5;
  }

  .field {
    margin-bottom: 1rem;
  }

  .field-label {
    display: block;
    font-size: 0.8125rem;
    color: var(--text-secondary);
    margin-bottom: 0.375rem;
    letter-spacing: 0.02em;
  }

  .field-input,
  .field-select {
    width: 100%;
    padding: 0.5rem 0.75rem;
    font-size: 0.875rem;
    font-family: inherit;
    color: var(--text-primary);
    background: var(--bg-input);
    border: 1px solid var(--border);
    border-radius: 6px;
    transition: border-color var(--transition), box-shadow var(--transition);
  }

  .field-input:focus,
  .field-select:focus {
    outline: none;
    border-color: var(--gold-dim);
    box-shadow: 0 0 0 2px rgba(196, 168, 114, 0.08);
  }

  .field-hint {
    display: block;
    font-size: 0.75rem;
    color: var(--text-muted);
    margin-top: 0.25rem;
  }

  .toggle-row {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    padding: 0.75rem 0;
    cursor: pointer;
    border-bottom: 1px solid var(--border);
  }

  .toggle-row:last-of-type {
    border-bottom: none;
  }

  .toggle-row input[type="checkbox"] {
    margin-top: 0.125rem;
    width: 1rem;
    height: 1rem;
    accent-color: var(--gold);
    flex-shrink: 0;
  }

  .toggle-label {
    font-size: 0.875rem;
    color: var(--text-primary);
    min-width: 5rem;
    flex-shrink: 0;
  }

  .toggle-desc {
    font-size: 0.8125rem;
    color: var(--text-muted);
    flex: 1;
  }

  .save-area {
    padding-top: 0.5rem;
  }

  .save-btn {
    padding: 0.625rem 1.5rem;
    font-size: 0.875rem;
    font-family: var(--font-heading);
    letter-spacing: 0.04em;
    color: var(--bg-primary);
    background: var(--gold);
    border: none;
    border-radius: 6px;
    cursor: pointer;
    transition: opacity var(--transition);
  }

  .save-btn:hover {
    opacity: 0.9;
  }

  .save-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .save-message {
    font-size: 0.8125rem;
    padding: 0.5rem 0;
    margin: 0;
  }

  .save-message.success {
    color: var(--gold);
  }

  .save-message.error {
    color: #e05252;
  }

  .save-hint {
    font-size: 0.75rem;
    color: var(--text-muted);
    margin-top: 0.5rem;
  }

  .setup-guide {
    margin: 0.5rem 0 1rem 1.75rem;
    padding: 1rem;
    background: var(--bg-input);
    border: 1px solid var(--border);
    border-left: 2px solid var(--gold-dim);
    border-radius: 6px;
  }

  .guide-title {
    font-family: var(--font-heading);
    font-size: 0.8125rem;
    font-weight: 400;
    color: var(--text-accent);
    letter-spacing: 0.04em;
    margin: 0 0 0.75rem;
  }

  .guide-steps {
    margin: 0;
    padding-left: 1.25rem;
    font-size: 0.8125rem;
    color: var(--text-secondary);
    line-height: 1.7;
  }

  .guide-steps li {
    margin-bottom: 0.5rem;
  }

  .guide-steps a {
    color: var(--gold);
    text-decoration: none;
    border-bottom: 1px solid var(--gold-dim);
  }

  .guide-steps a:hover {
    border-bottom-color: var(--gold);
  }

  .guide-steps code {
    font-family: var(--font-mono, 'JetBrains Mono', monospace);
    font-size: 0.75rem;
    padding: 0.125rem 0.375rem;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 3px;
    color: var(--gold);
  }

  .guide-code {
    display: block;
    margin: 0.5rem 0;
    padding: 0.625rem 0.75rem;
    font-family: var(--font-mono, 'JetBrains Mono', monospace);
    font-size: 0.75rem;
    line-height: 1.6;
    color: var(--text-secondary);
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 4px;
    overflow-x: auto;
    white-space: pre;
  }
</style>
