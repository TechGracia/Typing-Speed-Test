<script>
  const PARAGRAPHS = {
    easy: [
      'The quick brown fox jumps over the lazy dog.',
      'Typing is a useful skill to practice every day.',
      'Keep your fingers on the home row and look ahead.'
    ],
    medium: [
      'Svelte makes building reactive interfaces enjoyable and surprisingly simple.',
      'Practice every day to improve both your typing speed and accuracy.',
      'Focus on precision first and let speed follow naturally over time.'
    ],
    hard: [
      'Small consistent improvements over time create significant long-term growth in technical skills.',
      'Learning through projects is one of the fastest ways to deepen your development experience.',
      'Deliberate practice with immediate feedback accelerates mastery far beyond passive repetition.'
    ]
  };

  let dark = $state(false);
  let difficulty = $state('easy');
  let paragraph = $state('');
  let userInput = $state('');
  let timer = $state(60);
  let started = $state(false);
  let finished = $state(false);
  let finalWpm = $state(0);
  let finalAccuracy = $state(0);
  let bestScore = $state(0);
  let interval = $state(null);
  let themeMenuOpen = $state(false);

  if (typeof localStorage !== 'undefined') {
    bestScore = Number(localStorage.getItem('bestWPM')) || 0;
  }

  function getRandom(arr) {
    return arr[Math.floor(Math.random() * arr.length)];
  }

  paragraph = getRandom(PARAGRAPHS.easy);

  let correctChars = $derived(
    userInput.split('').filter((c, i) => c === paragraph[i]).length
  );
  let accuracy = $derived(
    userInput.length ? Math.round((correctChars / userInput.length) * 100) : 100
  );
  let elapsed = $derived(Math.max((60 - timer) / 60, 1 / 60));
  let wpm = $derived(Math.max(0, Math.round((correctChars / 5) / elapsed)));
  let progress = $derived(
    paragraph.length ? Math.round((userInput.length / paragraph.length) * 100) : 0
  );

  // BUG 1: Explicit per-character state — each index evaluated independently.
  // A wrong char at position N never infects position N+1.
  let chars = $derived(
    paragraph.split('').map((char, i) => {
      if (i < userInput.length) return userInput[i] === char ? 'correct' : 'wrong';
      if (i === userInput.length && !finished) return 'current';
      return '';
    })
  );

  $effect(() => {
    if (!finished && userInput.length > 0 && userInput === paragraph) finishTest();
  });

  function finishTest() {
    if (finished) return;
    clearInterval(interval);
    finished = true;
    finalWpm = wpm;
    finalAccuracy = accuracy; // BUG 4: snapshot — never reads live accuracy again
    if (finalWpm > bestScore) {
      bestScore = finalWpm;
      if (typeof localStorage !== 'undefined') localStorage.setItem('bestWPM', finalWpm);
    }
  }

  function startTimer() {
    if (started || finished) return;
    started = true;
    interval = setInterval(() => {
      if (timer > 1) { timer--; } else { timer = 0; finishTest(); }
    }, 1000);
  }

  function restart() {
    clearInterval(interval);
    userInput = '';
    timer = 60;
    started = false;
    finished = false;
    finalWpm = 0;
    finalAccuracy = 0;
    // Restart retries the SAME paragraph (paragraph is intentionally not reassigned here)
  }

  function newParagraph() {
    clearInterval(interval);
    userInput = '';
    timer = 60;
    started = false;
    finished = false;
    finalWpm = 0;
    finalAccuracy = 0;
    paragraph = getRandom(PARAGRAPHS[difficulty]);
  }

  function handleDifficulty(d) {
    difficulty = d;
    newParagraph();
  }

  function setTheme(isDark) {
    dark = isDark;
    themeMenuOpen = false;
  }

  let displayWpm = $state(0);
  let displayAcc = $state(0);

  // BUG 5: animate from frozen snapshots — displayAcc never reads live accuracy
  $effect(() => {
    if (finished) {
      const targetWpm = finalWpm;
      const targetAcc = finalAccuracy;
      const duration = 500;
      const start = performance.now();
      let raf;
      function tick(now) {
        const t = Math.min(1, (now - start) / duration);
        const ease = 1 - Math.pow(1 - t, 3);
        displayWpm = Math.round(targetWpm * ease);
        displayAcc = Math.round(targetAcc * ease);
        if (t < 1) raf = requestAnimationFrame(tick);
      }
      raf = requestAnimationFrame(tick);
      return () => cancelAnimationFrame(raf);
    } else {
      displayWpm = 0;
      displayAcc = 0;
    }
  });

  let inputEl = $state(null);

  function focusInput() {
    if (inputEl) inputEl.focus();
  }

  // Optional UX: ESC restarts (same paragraph), Enter submits when not finished
  function handleKeydown(e) {
    if (e.key === 'Escape') {
      e.preventDefault();
      restart();
      focusInput();
    } else if (e.key === 'Enter' && !finished) {
      e.preventDefault();
      finishTest();
    }
  }

  $effect(() => {
    // Autofocus the typing input whenever a fresh attempt becomes available
    if (!finished) focusInput();
  });

  function resetBestScore() {
    bestScore = 0;
    if (typeof localStorage !== 'undefined') localStorage.setItem('bestWPM', '0');
  }

  function spawnConfetti(node) {
    const colors = ['#E43D12','#EFB11D','#22C55E','#D6536D','#0FA4AF'];
    const timers = [];
    for (let i = 0; i < 40; i++) {
      const p = document.createElement('div');
      p.style.cssText = `
        position:absolute;left:${Math.random()*100}%;top:0;
        width:7px;height:7px;background:${colors[Math.floor(Math.random()*colors.length)]};
        border-radius:${Math.random()>0.5?'50%':'2px'};opacity:0;
        animation:conffall ${0.9+Math.random()*0.6}s ${Math.random()*0.5}s ease-in forwards;
      `;
      node.appendChild(p);
      // BUG 6: track every timer so destroy() can cancel them all
      const t = setTimeout(() => p.remove(), 2000);
      timers.push(t);
    }
    return {
      destroy() {
        timers.forEach(clearTimeout);
      }
    };
  }
</script>

<svelte:head>
  <title>typespeed</title>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet" />
</svelte:head>

<svelte:window onclick={() => themeMenuOpen = false} onkeydown={handleKeydown} />

<div class="app" class:dark>
  <div class="wrap">

    <!-- Header -->
    <header class="topbar">
      <div class="logo">
        <div class="logo-icon">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="4" width="20" height="16" rx="2"/><path d="M6 8h.01M10 8h.01M14 8h.01M18 8h.01M8 12h.01M12 12h.01M16 12h.01M7 16h10"/>
          </svg>
        </div>
        <span class="logo-title">typespeed</span>
      </div>

      <div class="theme-dropdown">
        <button class="theme-btn" onclick={(e) => { e.stopPropagation(); themeMenuOpen = !themeMenuOpen; }}>
          <span class="theme-icon">{dark ? '🌙' : '☀️'}</span>
          <span>{dark ? 'Dark' : 'Light'}</span>
          <svg class="chev" class:open={themeMenuOpen} width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"/></svg>
        </button>
        {#if themeMenuOpen}
          <div class="theme-menu">
            <button class="theme-option" class:active={!dark} onclick={() => setTheme(false)}>
              <span class="theme-icon">☀️</span> Light
            </button>
            <button class="theme-option" class:active={dark} onclick={() => setTheme(true)}>
              <span class="theme-icon">🌙</span> Dark
            </button>
          </div>
        {/if}
      </div>
    </header>

    <!-- Stats -->
    <div class="stats-grid">
      <div class="stat-card">
        <div class="stat-top">
          <div class="stat-icon stat-icon-time">
            <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2"><circle cx="12" cy="12" r="10"/><path d="M12 6v6l4 2"/></svg>
          </div>
          <span class="stat-label">time left</span>
        </div>
        <div class="stat-value stat-value-time">{timer}<span class="stat-unit">s</span></div>
        <svg class="sparkline" viewBox="0 0 100 24" preserveAspectRatio="none">
          <path d="M0,18 Q8,10 16,16 T32,12 T48,18 T64,8 T80,14 T97,4" fill="none" stroke="#F59E0B" stroke-width="1.5"/>
          <circle cx="97" cy="4" r="2.5" fill="#F59E0B"/>
        </svg>
        <div class="stat-progress"><div class="stat-progress-fill stat-progress-time" style="width: {Math.round((timer/60)*100)}%"></div></div>
      </div>

      <div class="stat-card">
        <div class="stat-top">
          <div class="stat-icon stat-icon-wpm">
            <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
          </div>
          <span class="stat-label">wpm</span>
        </div>
        <div class="stat-value stat-value-wpm">{finished ? finalWpm : wpm}</div>
        <svg class="sparkline" viewBox="0 0 100 24" preserveAspectRatio="none">
          <path d="M0,16 Q10,18 20,12 T40,14 T60,6 T80,10 T97,4" fill="none" stroke="#D6536D" stroke-width="1.5"/>
          <circle cx="97" cy="4" r="2.5" fill="#D6536D"/>
        </svg>
      </div>

      <div class="stat-card">
        <div class="stat-top">
          <div class="stat-icon stat-icon-acc">
            <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2"><circle cx="12" cy="12" r="10"/><circle cx="12" cy="12" r="6"/><circle cx="12" cy="12" r="2"/></svg>
          </div>
          <span class="stat-label">accuracy</span>
        </div>
        <div class="stat-value stat-value-acc">{accuracy}<span class="stat-unit">%</span></div>
        <svg class="sparkline" viewBox="0 0 100 24" preserveAspectRatio="none">
          <path d="M0,10 Q12,14 24,8 T48,10 T72,6 T97,8" fill="none" stroke="#22C55E" stroke-width="1.5"/>
          <circle cx="97" cy="8" r="2.5" fill="#22C55E"/>
        </svg>
      </div>

      <div class="stat-card">
        <div class="stat-top">
          <div class="stat-icon stat-icon-best">
            <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"/></svg>
          </div>
          <span class="stat-label">best score</span>
          <button class="stat-reset" onclick={resetBestScore} aria-label="Reset best score" title="Reset best score">
            <svg width="11" height="11" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 .49-4.2"/></svg>
          </button>
        </div>
        <div class="stat-value stat-value-best">{bestScore}</div>
        <svg class="sparkline" viewBox="0 0 100 24" preserveAspectRatio="none">
          <path d="M0,18 Q10,16 20,18 T40,10 T60,14 T80,6 T97,6" fill="none" stroke="#D6536D" stroke-width="1.5"/>
          <circle cx="97" cy="6" r="2.5" fill="#D6536D"/>
        </svg>
      </div>
    </div>

    <!-- Difficulty row -->
    <div class="diff-row">
      <div class="seg-wrap">
        <span class="seg-label">DIFFICULTY</span>
        <div class="seg">
          <!-- Animated sliding active indicator, position driven by --active-index -->
          <div class="seg-indicator" style="--active-index: {['easy','medium','hard'].indexOf(difficulty)}"></div>
          {#each ['easy','medium','hard'] as d}
            <button
              class="seg-btn"
              class:active={difficulty === d}
              onclick={() => handleDifficulty(d)}
            >{d.charAt(0).toUpperCase() + d.slice(1)}</button>
          {/each}
        </div>
      </div>
      <button class="new-para-btn" onclick={newParagraph}>
        <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><polyline points="16 3 21 3 21 8"/><line x1="4" y1="20" x2="21" y2="3"/><polyline points="21 16 21 21 16 21"/><line x1="15" y1="15" x2="21" y2="21"/><line x1="4" y1="4" x2="9" y2="9"/></svg>
        New Paragraph
      </button>
    </div>

    <!-- Progress bar -->
    <div class="prog-row">
      <div class="prog-track" role="progressbar" aria-valuenow={progress} aria-valuemin="0" aria-valuemax="100">
        <div class="prog-fill" style="width: {progress}%"></div>
      </div>
      <span class="prog-pct">{progress}% Complete</span>
    </div>

    <!-- Paragraph display -->
    <div class="para-zone">
      <p class="para-text">
        {#each paragraph.split('') as char, i}
          <span
            class="ch"
            class:correct={chars[i] === 'correct'}
            class:wrong={chars[i] === 'wrong'}
            class:current={chars[i] === 'current'}
          >{char === ' ' ? '\u00A0' : char}</span>
        {/each}
      </p>
    </div>

    <!-- Typing input -->
    {#if !finished}
      <div class="input-wrap">
        <textarea
          bind:value={userInput}
          bind:this={inputEl}
          oninput={startTimer}
          placeholder="Start typing here..."
          maxlength={paragraph.length}
          spellcheck="false"
          autocomplete="off"
          autocorrect="off"
          autocapitalize="off"
          aria-label="Typing input"
          class="input-field"
        ></textarea>
        <div class="input-kbd-icon">
          <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="4" width="20" height="16" rx="2"/><path d="M6 8h.01M10 8h.01M14 8h.01M18 8h.01M8 12h.01M12 12h.01M16 12h.01M7 16h10"/>
          </svg>
        </div>
      </div>
    {/if}

    <!-- Buttons -->
    <div class="btns">
      {#if !finished}
        <button class="btn-submit" onclick={finishTest}>
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.3" stroke-linecap="round" stroke-linejoin="round"><line x1="22" y1="2" x2="11" y2="13"/><polygon points="22 2 15 22 11 13 2 9 22 2"/></svg>
          Submit
        </button>
      {/if}
      <button class="btn-restart" onclick={restart}>
        <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.3" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 .49-4.2"/></svg>
        Restart
      </button>
    </div>

    <!-- Help cards -->
    <div class="help-row">
      <div class="help-card">
        <div class="help-icon help-icon-blue">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><polygon points="5 3 19 12 5 21 5 3"/></svg>
        </div>
        <div class="help-text">
          <span class="help-title">Start typing</span>
          <span class="help-desc">The timer starts as soon as you type</span>
        </div>
      </div>
      <div class="help-card">
        <div class="help-icon help-icon-green">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2"><circle cx="12" cy="12" r="10"/><circle cx="12" cy="12" r="6"/><circle cx="12" cy="12" r="2"/></svg>
        </div>
        <div class="help-text">
          <span class="help-title">Stay accurate</span>
          <span class="help-desc">Accuracy affects your WPM score</span>
        </div>
      </div>
      <div class="help-card">
        <div class="help-icon help-icon-pink">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
        </div>
        <div class="help-text">
          <span class="help-title">Finish strong</span>
          <span class="help-desc">Time runs out at 0 seconds</span>
        </div>
      </div>
    </div>

  </div>

  <!-- Result modal -->
  {#if finished}
    <div class="modal-backdrop" use:spawnConfetti>
      <div class="modal">
        <div class="modal-title">Test Complete</div>
        <div class="modal-nums">
          <div class="modal-stat">
            <span class="modal-val">{displayWpm}</span>
            <span class="modal-lbl">WPM</span>
          </div>
          <div class="modal-div"></div>
          <div class="modal-stat">
            <span class="modal-val modal-val-acc">{displayAcc}%</span>
            <span class="modal-lbl">Accuracy</span>
          </div>
        </div>
        {#if finalWpm > 0 && finalWpm === bestScore}
          <div class="modal-badge">🏆 New personal best!</div>
        {/if}
        <div class="modal-btns">
          <button class="btn-restart" onclick={restart}>
            <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.3" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 .49-4.2"/></svg>
            Try again
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>


<style>
  /* ════════════════════════════════════════════════
     TOKENS — Light mode (warm, high-contrast)
  ════════════════════════════════════════════════ */
  .app {
    --bg:        #F2EAE0;
    --surface:   #FFFFFF;
    --surface2:  #F7F1EB;
    --border:    #DDD4C8;
    --primary:   #E43D12;
    --secondary: #D6536D;
    --accent:    #EFB11D;
    --success:   #22C55E;
    --text:      #1E1B18;
    --muted:     #7A726C;
    --faint:     #A89F98;
    --correct:   #22C55E;
    --wrong:     #E43D12;
    --caret-clr: #EFB11D;
    --mono:      'JetBrains Mono', monospace;
    --ui:        'Inter', -apple-system, sans-serif;
    --shadow-sm: 0 1px 3px rgba(46,42,39,.10), 0 1px 2px rgba(46,42,39,.07);
    --shadow-md: 0 4px 20px rgba(46,42,39,.14), 0 2px 8px rgba(46,42,39,.09);
    --shadow-lg: 0 12px 40px rgba(46,42,39,.18), 0 4px 12px rgba(46,42,39,.10);

    min-height: 100vh;
    background: var(--bg);
    color: var(--text);
    font-family: var(--ui);
    transition: background .25s ease, color .2s ease;
    padding-bottom: 48px;
  }

  /* ════════════════════════════════════════════════
     TOKENS — Dark mode (deep contrast)
  ════════════════════════════════════════════════ */
  .app.dark {
    --bg:        #0a1416;
    --surface:   #142024;
    --surface2:  #0f1a1e;
    --border:    rgba(175,221,229,.11);
    --primary:   #0FA4AF;
    --secondary: #0FA4AF;
    --accent:    #24D6D1;
    --success:   #24D6D1;
    --text:      #C8E8ED;
    --muted:     #6b9399;
    --faint:     #3a6068;
    --correct:   #5DCAA5;
    --wrong:     #F0997B;
    --caret-clr: #24D6D1;
    --shadow-sm: 0 1px 3px rgba(0,0,0,.45);
    --shadow-md: 0 8px 28px rgba(0,0,0,.55);
    --shadow-lg: 0 20px 60px rgba(0,0,0,.65);
    background-image: radial-gradient(ellipse 1200px 600px at 50% -10%,
      rgba(15,164,175,.08), transparent 60%);
  }

  :global(*) { box-sizing: border-box; margin: 0; padding: 0; }
  :global(body) { background: var(--bg); }

  /* ════════════════════════════════════════════════
     LAYOUT
  ════════════════════════════════════════════════ */
  .wrap {
    max-width: 760px;
    margin: 0 auto;
    padding: 24px 24px 0;
  }

  /* ════════════════════════════════════════════════
     HEADER
  ════════════════════════════════════════════════ */
  .topbar {
    display: flex; align-items: center; justify-content: space-between;
    margin-bottom: 20px;
  }
  .logo { display: flex; align-items: center; gap: 10px; }
  .logo-icon {
    width: 32px; height: 32px;
    background: color-mix(in srgb, var(--primary) 12%, transparent);
    border-radius: 8px;
    display: flex; align-items: center; justify-content: center;
    color: var(--primary);
  }
  .dark .logo-icon { color: var(--accent); background: color-mix(in srgb, var(--accent) 14%, transparent); }
  .logo-title { font-size: 16px; font-weight: 700; letter-spacing: -0.3px; color: var(--text); }

  .theme-dropdown { position: relative; }
  .theme-btn {
    display: flex; align-items: center; gap: 8px;
    font-family: var(--ui); font-size: 13px; font-weight: 500;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 10px; padding: 9px 15px;
    color: var(--text); cursor: pointer;
    box-shadow: var(--shadow-sm);
    transition: border-color .15s, box-shadow .15s, transform .15s;
  }
  .theme-btn:hover {
    border-color: color-mix(in srgb, var(--text) 25%, transparent);
    transform: translateY(-1px);
    box-shadow: var(--shadow-md);
  }
  .theme-icon { font-size: 14px; line-height: 1; display: inline-block; transition: transform .35s ease; }
  .theme-btn:hover .theme-icon { transform: rotate(20deg); }
  .chev { transition: transform .18s ease; color: var(--muted); }
  .chev.open { transform: rotate(180deg); }

  .theme-menu {
    position: absolute; top: calc(100% + 6px); right: 0;
    background: var(--surface); border: 1px solid var(--border);
    border-radius: 10px; box-shadow: var(--shadow-md);
    padding: 4px; min-width: 120px; z-index: 50;
    display: flex; flex-direction: column; gap: 2px;
    animation: dropdown-in .15s ease;
  }
  @keyframes dropdown-in { from { opacity: 0; transform: translateY(-4px); } to { opacity: 1; transform: translateY(0); } }
  .theme-option {
    display: flex; align-items: center; gap: 8px;
    font-family: var(--ui); font-size: 13px; font-weight: 500;
    padding: 7px 10px; border: none; border-radius: 7px;
    background: none; color: var(--muted); cursor: pointer;
    text-align: left; transition: background .12s, color .12s;
  }
  .theme-option:hover { background: var(--surface2); color: var(--text); }
  .theme-option.active { color: var(--text); background: var(--surface2); font-weight: 600; }

  /* ════════════════════════════════════════════════
     STATS GRID
  ════════════════════════════════════════════════ */
  .stats-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 14px;
  }
  .stat-card {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 14px;
    padding: 12px 15px 11px;
    box-shadow: var(--shadow-sm);
    position: relative;
    overflow: hidden;
    transition: transform .16s ease, box-shadow .16s ease;
  }
  .dark .stat-card {
    background: var(--surface);
    border-color: rgba(175,221,229,.13);
    box-shadow: 0 2px 12px rgba(0,0,0,.5), inset 0 1px 0 rgba(175,221,229,.06);
  }
  .stat-card:hover { transform: translateY(-2px); box-shadow: var(--shadow-md); }

  .stat-top { display: flex; align-items: center; gap: 8px; margin-bottom: 9px; }
  .stat-icon {
    width: 26px; height: 26px; border-radius: 7px;
    display: flex; align-items: center; justify-content: center;
    flex-shrink: 0;
  }
  .stat-icon-time  { background: rgba(245,158,11,.14); color: #F59E0B; }
  .stat-icon-wpm   { background: color-mix(in srgb, var(--secondary) 14%, transparent); color: var(--secondary); }
  .stat-icon-acc   { background: rgba(34,197,94,.14); color: #22C55E; }
  .stat-icon-best  { background: color-mix(in srgb, var(--secondary) 14%, transparent); color: var(--secondary); }
  .dark .stat-icon-acc  { background: rgba(45,212,191,.14); color: var(--accent); }
  .dark .stat-icon-best { background: rgba(45,212,191,.14); color: var(--accent); }

  .stat-label {
    font-size: 11px; font-weight: 600; text-transform: uppercase;
    letter-spacing: .6px; color: var(--muted);
  }
  .stat-value {
    font-family: var(--mono);
    font-size: 32px; font-weight: 700;
    letter-spacing: -1px; line-height: 1;
    margin-bottom: 7px; min-width: 1.2ch;
  }
  .stat-value-time { color: #F59E0B; }
  .stat-value-wpm  { color: var(--secondary); }
  .stat-value-acc  { color: #22C55E; }
  .dark .stat-value-acc  { color: var(--accent); }
  .stat-value-best { color: var(--secondary); }
  .dark .stat-value-best { color: var(--accent); }
  .stat-unit { font-size: 14px; font-weight: 500; opacity: .60; margin-left: 1px; }

  /* Sparkline — contained, no overflow bleed */
  .sparkline {
    display: block;
    width: 100%;
    height: 20px;
    opacity: 0.80;
    overflow: visible;
  }
  .stat-card > .sparkline { clip-path: inset(0 0 0 0); }

  .stat-progress {
    height: 3px; border-radius: 99px; overflow: hidden;
    background: color-mix(in srgb, var(--text) 8%, transparent);
    margin-top: 7px;
  }
  .stat-progress-fill { height: 100%; border-radius: 99px; transition: width .25s ease; }
  .stat-progress-time { background: #F59E0B; }

  .stat-reset {
    position: absolute; top: 10px; right: 10px;
    width: 20px; height: 20px; border: none; border-radius: 6px;
    background: transparent; color: var(--faint); cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    opacity: 0; transition: opacity .15s, background .15s, color .15s;
  }
  .stat-card:hover .stat-reset { opacity: 1; }
  .stat-reset:hover { background: color-mix(in srgb, var(--text) 8%, transparent); color: var(--text); }

  /* ════════════════════════════════════════════════
     DIFFICULTY ROW
  ════════════════════════════════════════════════ */
  .diff-row {
    display: flex; align-items: center; justify-content: space-between;
    margin-bottom: 10px; gap: 12px; flex-wrap: wrap;
  }
  .seg-wrap { display: flex; align-items: center; gap: 12px; }
  .seg-label { font-size: 11px; font-weight: 700; letter-spacing: 1px; color: var(--muted); }
  .seg {
    display: flex; position: relative;
    background: transparent; border: none;
    border-radius: 10px; padding: 3px; gap: 2px;
  }
  .seg-indicator {
    position: absolute; top: 3px; left: 3px;
    height: calc(100% - 6px);
    width: calc((100% - 6px - 4px) / 3);
    border-radius: 7px;
    background: linear-gradient(135deg, var(--primary), var(--secondary));
    box-shadow: 0 4px 14px color-mix(in srgb, var(--primary) 40%, transparent),
                0 0 0 1px color-mix(in srgb, var(--primary) 15%, transparent);
    transform: translateX(calc(var(--active-index) * (100% + 2px)));
    transition: transform .28s cubic-bezier(0.34,1.56,0.64,1);
    z-index: 0;
  }
  .dark .seg-indicator {
    background: linear-gradient(135deg, #0FA4AF, #24D6D1);
    box-shadow: 0 4px 16px rgba(15,164,175,.45), 0 0 0 1px rgba(36,214,209,.2);
  }
  .seg-btn {
    font-family: var(--ui); font-size: 13px; font-weight: 500;
    padding: 6px 18px; border: none; border-radius: 7px;
    background: none; color: var(--muted); cursor: pointer;
    transition: color .15s; position: relative; z-index: 1;
  }
  .seg-btn.active { color: #fff; background: none; box-shadow: none; }
  .dark .seg-btn.active { color: #00191c; }
  .seg-btn:not(.active):hover { color: var(--text); }

  .new-para-btn {
    display: flex; align-items: center; gap: 7px;
    font-family: var(--ui); font-size: 13px; font-weight: 500;
    background: color-mix(in srgb, var(--primary) 8%, transparent);
    border: none; border-radius: 9px; padding: 8px 16px;
    color: var(--primary); cursor: pointer;
    transition: background .15s, transform .12s;
  }
  .dark .new-para-btn { color: var(--accent); background: color-mix(in srgb, var(--accent) 10%, transparent); }
  .new-para-btn:hover {
    background: color-mix(in srgb, var(--primary) 14%, transparent);
    transform: translateY(-1px);
  }
  .dark .new-para-btn:hover { background: color-mix(in srgb, var(--accent) 16%, transparent); }

  /* ════════════════════════════════════════════════
     PROGRESS BAR
  ════════════════════════════════════════════════ */
  .prog-row {
    display: flex; align-items: center; gap: 14px;
    margin-bottom: 14px;
  }
  .prog-track {
    flex: 1; height: 5px;
    background: color-mix(in srgb, var(--muted) 16%, transparent);
    border-radius: 99px; overflow: hidden;
  }
  .prog-fill {
    height: 100%;
    background: linear-gradient(90deg, var(--primary), var(--secondary));
    border-radius: 99px; transition: width .12s ease;
  }
  .dark .prog-fill { background: var(--accent); }
  .prog-pct { font-size: 12px; font-weight: 500; color: var(--muted); white-space: nowrap; }

  /* ════════════════════════════════════════════════
     PARAGRAPH ZONE
  ════════════════════════════════════════════════ */
  .para-zone {
    min-height: 220px;
    max-height: 260px;
    display: flex;
    align-items: center;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 16px;
    padding: 28px 34px;
    margin-bottom: 16px;
    box-shadow: var(--shadow-sm);
    position: relative;
    overflow: hidden;
  }
  .dark .para-zone {
    border-color: rgba(175,221,229,.13);
    box-shadow: 0 2px 16px rgba(0,0,0,.55), inset 0 1px 0 rgba(175,221,229,.05);
  }
  /* Ambient spotlight */
  .para-zone::before {
    content: '';
    position: absolute; top: -60%; left: 50%;
    width: 140%; height: 220%;
    transform: translateX(-50%);
    background: radial-gradient(ellipse at center,
      color-mix(in srgb, var(--caret-clr) 7%, transparent) 0%, transparent 60%);
    pointer-events: none;
  }

  .para-text {
    font-family: var(--mono);
    font-size: 34px;
    line-height: 1.65;
    letter-spacing: -0.025em;
    opacity: 0.82;
    color: var(--faint);
    max-width: 22ch;
    word-break: break-word;
    user-select: none;
    position: relative;
  }

  /* ── Character states ── */
  .ch { transition: color 0.08s ease; }
  .ch.correct { color: var(--correct); font-weight: 500; opacity: 1; }
  .ch.wrong   { color: var(--wrong); text-decoration: underline wavy; text-underline-offset: 4px; opacity: 1; }

  /* Current character — rounded, glowing, pulsing */
  .ch.current {
    color: var(--text);
    background: color-mix(in srgb, var(--caret-clr) 28%, transparent);
    border-radius: 4px;
    padding: 0 1px;
    box-shadow:
      0 0 0 1.5px color-mix(in srgb, var(--caret-clr) 50%, transparent),
      0 0 14px color-mix(in srgb, var(--caret-clr) 40%, transparent),
      0 0 28px color-mix(in srgb, var(--caret-clr) 18%, transparent);
    animation: cursorPulse 1.25s ease-in-out infinite;
    opacity: 1;
  }
  @keyframes cursorPulse {
    0%, 100% {
      box-shadow:
        0 0 0 1.5px color-mix(in srgb, var(--caret-clr) 50%, transparent),
        0 0 10px color-mix(in srgb, var(--caret-clr) 35%, transparent),
        0 0 22px color-mix(in srgb, var(--caret-clr) 12%, transparent);
    }
    50% {
      box-shadow:
        0 0 0 2px color-mix(in srgb, var(--caret-clr) 65%, transparent),
        0 0 18px color-mix(in srgb, var(--caret-clr) 50%, transparent),
        0 0 36px color-mix(in srgb, var(--caret-clr) 22%, transparent);
    }
  }

  /* Blinking caret underline (fallback visual) */
  .ch.current::after {
    content: '';
    position: absolute; bottom: -2px; left: 0; right: 0;
    height: 2px; border-radius: 3px;
    animation: cursor-blink 1s ease-in-out infinite;
  }
  @keyframes cursor-blink {
    0%, 100% { background: color-mix(in srgb, var(--caret-clr) 25%, transparent); }
    50%       { background: color-mix(in srgb, var(--caret-clr) 60%, transparent); }
  }

  /* ════════════════════════════════════════════════
     INPUT
  ════════════════════════════════════════════════ */
  .input-wrap { position: relative; margin-bottom: 16px; }
  .input-field {
    display: block; width: 100%;
    height: 120px;
    font-family: var(--mono); font-size: 17px;
    padding: 20px 52px 20px 20px;
    resize: none;
    background: var(--surface);
    border: 1.5px solid var(--border);
    border-radius: 14px;
    color: var(--text); outline: none;
    transition: border-color 0.15s, box-shadow 0.15s, background 0.15s;
    caret-color: var(--caret-clr);
    box-shadow: var(--shadow-sm);
    line-height: 1.6;
  }
  .dark .input-field {
    background: var(--surface2);
    border-color: rgba(175,221,229,.12);
    box-shadow: 0 2px 12px rgba(0,0,0,.5), inset 0 1px 0 rgba(175,221,229,.04);
  }
  .input-field::placeholder { color: var(--faint); font-style: italic; font-size: 13px; }

  /* Strong premium focus ring */
  .input-field:focus {
    border-color: var(--primary);
    background: color-mix(in srgb, var(--surface) 96%, var(--primary));
    box-shadow:
      0 0 0 3px color-mix(in srgb, var(--primary) 18%, transparent),
      0 0 0 6px color-mix(in srgb, var(--primary) 6%, transparent),
      var(--shadow-md);
  }
  .dark .input-field:focus {
    border-color: var(--accent);
    background: color-mix(in srgb, var(--surface2) 96%, var(--accent));
    box-shadow:
      0 0 0 3px color-mix(in srgb, var(--accent) 22%, transparent),
      0 0 0 6px color-mix(in srgb, var(--accent) 8%, transparent),
      var(--shadow-md);
  }

  .input-kbd-icon {
    position: absolute; right: 14px; top: 14px;
    width: 28px; height: 28px; border-radius: 7px;
    background: var(--surface2); border: 1px solid var(--border);
    display: flex; align-items: center; justify-content: center;
    color: var(--muted); pointer-events: none;
  }

  /* ════════════════════════════════════════════════
     BUTTONS
  ════════════════════════════════════════════════ */
  .btns { display: flex; gap: 10px; align-items: center; margin-bottom: 20px; }

  /* Submit — large, dominant, primary CTA */
  .btn-submit {
    display: inline-flex; align-items: center; gap: 9px;
    font-family: var(--ui); font-size: 15px; font-weight: 700;
    padding: 14px 36px;
    border: none; border-radius: 11px;
    background: linear-gradient(135deg, var(--primary) 0%, color-mix(in srgb, var(--primary) 70%, var(--secondary)) 100%);
    color: #fff; cursor: pointer;
    box-shadow:
      0 4px 16px color-mix(in srgb, var(--primary) 40%, transparent),
      0 1px 0 rgba(255,255,255,.12) inset;
    transition: transform 0.14s, box-shadow .14s, filter .12s;
    letter-spacing: -0.01em;
  }
  .dark .btn-submit {
    color: #00191c;
    background: linear-gradient(135deg, var(--accent), #0FA4AF);
    box-shadow:
      0 4px 18px color-mix(in srgb, var(--accent) 40%, transparent),
      0 1px 0 rgba(255,255,255,.10) inset;
  }
  .btn-submit:hover {
    transform: translateY(-2px);
    filter: brightness(1.06);
    box-shadow:
      0 8px 28px color-mix(in srgb, var(--primary) 50%, transparent),
      0 1px 0 rgba(255,255,255,.12) inset;
  }
  .dark .btn-submit:hover {
    box-shadow:
      0 8px 28px color-mix(in srgb, var(--accent) 50%, transparent),
      0 1px 0 rgba(255,255,255,.10) inset;
  }
  .btn-submit:active { transform: translateY(0) scale(.98); filter: brightness(.98); }

  /* Restart — small, quiet, secondary */
  .btn-restart {
    display: inline-flex; align-items: center; gap: 6px;
    font-family: var(--ui); font-size: 12px; font-weight: 500;
    padding: 9px 16px;
    border: 1.5px solid var(--border);
    border-radius: 9px;
    background: transparent;
    color: var(--muted); cursor: pointer;
    transition: color .14s, border-color .14s, transform .12s, box-shadow .12s;
  }
  .btn-restart:hover {
    color: var(--text);
    border-color: color-mix(in srgb, var(--text) 28%, transparent);
    transform: translateY(-1px);
    box-shadow: var(--shadow-sm);
  }
  .btn-restart:active { transform: translateY(0) scale(.98); }

  /* ════════════════════════════════════════════════
     HELP ROW
  ════════════════════════════════════════════════ */
  .help-row {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 14px;
    display: flex;
    box-shadow: var(--shadow-sm);
    overflow: hidden;
  }
  .dark .help-row {
    border-color: rgba(175,221,229,.10);
    box-shadow: 0 2px 12px rgba(0,0,0,.45);
  }
  .help-card {
    flex: 1; display: flex; align-items: center; gap: 12px;
    padding: 16px 18px;
    border-right: 1px solid var(--border);
  }
  .help-card:last-child { border-right: none; }
  .help-icon {
    width: 28px; height: 28px; border-radius: 8px; flex-shrink: 0;
    display: flex; align-items: center; justify-content: center;
  }
  .help-icon-blue  { background: color-mix(in srgb, #0FA4AF 16%, transparent); color: #0FA4AF; }
  .help-icon-green { background: rgba(34,197,94,.16); color: #22C55E; }
  .help-icon-pink  { background: color-mix(in srgb, var(--secondary) 16%, transparent); color: var(--secondary); }
  .dark .help-icon-green { color: var(--accent); background: rgba(45,212,191,.16); }
  .help-text { display: flex; flex-direction: column; gap: 2px; min-width: 0; }
  .help-title { font-size: 13px; font-weight: 600; color: var(--text); }
  .help-desc  { font-size: 11px; color: var(--muted); }

  /* ════════════════════════════════════════════════
     RESULT MODAL
  ════════════════════════════════════════════════ */
  .modal-backdrop {
    position: fixed; inset: 0;
    background: color-mix(in srgb, var(--bg) 80%, transparent);
    backdrop-filter: blur(8px);
    display: flex; align-items: center; justify-content: center;
    z-index: 100;
    animation: fadein 0.2s ease;
  }
  @keyframes fadein { from { opacity: 0; } to { opacity: 1; } }

  .modal {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 20px;
    padding: 38px 52px;
    text-align: center;
    min-width: 320px;
    position: relative; overflow: hidden;
    box-shadow: var(--shadow-lg);
    animation: pop 0.35s cubic-bezier(0.34,1.56,0.64,1);
  }
  .dark .modal {
    border-color: rgba(175,221,229,.14);
    box-shadow: 0 32px 80px rgba(0,0,0,.75), 0 0 0 1px rgba(175,221,229,.06);
  }
  @keyframes pop {
    from { opacity: 0; transform: scale(0.9) translateY(12px); }
    to   { opacity: 1; transform: scale(1) translateY(0); }
  }
  .modal-title {
    font-size: 11px; font-weight: 700; color: var(--muted);
    margin-bottom: 22px; letter-spacing: 1px; text-transform: uppercase;
  }
  .modal-nums { display: flex; align-items: center; justify-content: center; gap: 32px; margin-bottom: 18px; }
  .modal-stat { display: flex; flex-direction: column; align-items: center; gap: 4px; }
  .modal-val {
    font-family: var(--mono); font-size: 52px; font-weight: 700;
    letter-spacing: -2px; line-height: 1;
    background: linear-gradient(135deg, var(--primary), var(--secondary));
    -webkit-background-clip: text; background-clip: text;
    -webkit-text-fill-color: transparent; color: transparent;
  }
  .dark .modal-val { background: var(--accent); -webkit-background-clip: text; background-clip: text; }
  .modal-val-acc {
    background: linear-gradient(135deg, #22C55E, #16A34A);
    -webkit-background-clip: text; background-clip: text;
  }
  .dark .modal-val-acc { background: var(--accent); -webkit-background-clip: text; background-clip: text; }
  .modal-lbl { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; color: var(--muted); }
  .modal-div { width: 1px; height: 48px; background: var(--border); }
  .modal-badge {
    display: inline-block; font-size: 12px; font-weight: 600;
    color: var(--secondary);
    background: color-mix(in srgb, var(--secondary) 14%, transparent);
    border-radius: 99px; padding: 5px 14px; margin-bottom: 22px;
  }
  .dark .modal-badge { color: var(--accent); background: color-mix(in srgb, var(--accent) 14%, transparent); }
  .modal-btns { display: flex; justify-content: center; }

  /* confetti */
  @keyframes conffall {
    0%   { opacity: 1; transform: translateY(-10px) rotate(0deg); }
    100% { opacity: 0; transform: translateY(180px) rotate(360deg); }
  }

  /* ════════════════════════════════════════════════
     RESPONSIVE
  ════════════════════════════════════════════════ */
  @media (max-width: 700px) {
    .stats-grid { grid-template-columns: repeat(2, 1fr); gap: 10px; }
    .para-text   { font-size: 26px; max-width: none; }
    .para-zone   { padding: 22px 20px; }
    .modal       { padding: 28px 24px; min-width: 0; width: calc(100vw - 48px); }
    .modal-val   { font-size: 40px; }
    .help-row    { flex-direction: column; }
    .help-card   { border-right: none; border-bottom: 1px solid var(--border); }
    .help-card:last-child { border-bottom: none; }
    .diff-row    { flex-direction: column; align-items: stretch; }
    .new-para-btn { justify-content: center; }
  }

  @media (prefers-reduced-motion: reduce) {
    .ch.current { animation: none; }
    .modal, .modal-backdrop { animation: none; }
  }
</style>