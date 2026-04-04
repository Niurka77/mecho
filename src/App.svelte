<script>
  import { onMount, onDestroy } from 'svelte';
  import { fly } from 'svelte/transition';
  import { supabase } from './lib/supabaseClient';
  import Post from './lib/Post.svelte';

  let posts = [];
  let loading = true;
  let textInput = '';
  let mediaFile = null;
  let mediaPreview = null;
  let fileInputRef;
  let textareaRef;
  let scrollY = 0;
  let channel;
  
  let selectedColor = '#1a1c1e';
  let selectedMood = null;
  let isUploading = false;
  let showConfetti = false;
  let todayCount = 0;
  let popSound;
  
  // Toggle dark mode (por defecto: activado)
  let darkMode = true;
  
  // MOODS DISPONIBLES - Colores adaptados al tema dark
  const moods = [
    { emoji: '✿', label: 'Feliz', color: '#2a2c2e', textColor: '#F4C2C2', border: '#F4C2C2' },
    { emoji: '☁', label: 'Tranquilo', color: '#2a2c2e', textColor: '#A8C3D6', border: '#A8C3D6' },
    { emoji: '⚡', label: 'Energético', color: '#2a2c2e', textColor: '#E8D5A0', border: '#E8D5A0' },
    { emoji: '❀', label: 'En paz', color: '#2a2c2e', textColor: '#A8B8A0', border: '#A8B8A0' },
    { emoji: '✦', label: 'Melancólico', color: '#2a2c2e', textColor: '#C8A8D6', border: '#C8A8D6' },
    { emoji: '★', label: 'Motivado', color: '#2a2c2e', textColor: '#D6A8B8', border: '#D6A8B8' }
  ];

  onMount(async () => {
    // Cargar preferencia de modo oscuro desde localStorage
    const savedMode = localStorage.getItem('mecho-darkmode');
    if (savedMode !== null) {
      darkMode = savedMode === 'true';
    }
    applyTheme();
    
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) console.error('Error:', error);
    else posts = data || [];
    loading = false;
    updateTodayCount();

    channel = supabase
      .channel('mecho-realtime')
      .on('postgres_changes', 
        { event: 'INSERT', schema: 'public', table: 'posts' }, 
        payload => {
          posts = [payload.new, ...posts];
          updateTodayCount();
          if (popSound) popSound.play();
        }
      )
      .subscribe();

    window.addEventListener('scroll', handleScroll);
  });

  onDestroy(() => {
    if (channel) supabase.removeChannel(channel);
    window.removeEventListener('scroll', handleScroll);
  });

  // Aplicar tema al body
  function applyTheme() {
    document.body.classList.toggle('light-mode', !darkMode);
    localStorage.setItem('mecho-darkmode', String(darkMode));
  }
  
  function toggleDarkMode() {
    darkMode = !darkMode;
    applyTheme();
  }

  function handleScroll() { scrollY = window.scrollY; }
  
  function updateTodayCount() {
    const today = new Date().toDateString();
    todayCount = posts.filter(p => new Date(p.created_at).toDateString() === today).length;
  }

  function handleFileChange(event) {
    mediaFile = event.target.files[0];
    if (mediaFile) {
      const reader = new FileReader();
      reader.onload = e => { mediaPreview = e.target.result; };
      reader.readAsDataURL(mediaFile);
    } else { mediaPreview = null; }
  }

  function selectMood(mood) {
    selectedMood = mood;
    selectedColor = mood.color;
  }

  function openEmojiPicker() {
    if (textareaRef) {
      textareaRef.focus();
    }
  }

  function createConfetti() {
    showConfetti = true;
    const colors = ['#F4C2C2', '#A8C3D6', '#A8B8A0', '#E8D5A0', '#C8A8D6'];
    for (let i = 0; i < 40; i++) {
      const confetti = document.createElement('div');
      confetti.className = 'confetti-pixel';
      confetti.style.left = Math.random() * 100 + '%';
      confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
      confetti.style.width = Math.random() * 6 + 4 + 'px';
      confetti.style.height = Math.random() * 6 + 4 + 'px';
      confetti.style.animationDuration = Math.random() * 2 + 1 + 's';
      document.body.appendChild(confetti);
      setTimeout(() => confetti.remove(), 3000);
    }
    setTimeout(() => { showConfetti = false; }, 3000);
  }

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✿ escribe algo');
      return;
    }
    if (!selectedMood) {
      alert('✿ selecciona un mood');
      return;
    }

    isUploading = true;
    let currentMediaUrl = null;

    if (mediaFile) {
      const fileExt = mediaFile.name.split('.').pop();
      const safeName = `${Date.now()}_${Math.random().toString(36).slice(2, 8)}.${fileExt}`;
      const datePath = new Date().toISOString().split('T')[0];
      const filePath = `posts/${datePath}/${safeName}`;

      const { error: uploadError } = await supabase.storage
        .from('mecho-media')
        .upload(filePath, mediaFile, { cacheControl: '3600' });

      if (uploadError) {
        console.error('Error:', uploadError);
        alert('no se pudo subir');
        isUploading = false;
        return;
      }
      const { data: { publicUrl } } = supabase.storage.from('mecho-media').getPublicUrl(filePath);
      currentMediaUrl = publicUrl;
    }

    let postType = 'note';
    if (mediaFile) {
      if (mediaFile.type.startsWith('image')) postType = 'image';
      else if (mediaFile.type.startsWith('video')) postType = 'video';
      else if (mediaFile.type.startsWith('audio')) postType = 'audio';
    }

    const newPostData = {
      text: textInput.trim() || null,
      media_url: currentMediaUrl,
      color: selectedColor,
      mood: selectedMood.label,
      mood_emoji: selectedMood.emoji,
      mood_border: selectedMood.border,
      type: postType,
      likes: 0
    };

    const { error: dbError } = await supabase.from('posts').insert([newPostData]);
    if (dbError) {
      console.error('Error BD:', dbError);
      alert('algo salió mal');
    } else {
      createConfetti();
      if (popSound) popSound.play();
      textInput = ''; 
      mediaFile = null; 
      mediaPreview = null;
      selectedMood = null;
      selectedColor = '#1a1c1e';
      if (fileInputRef) fileInputRef.value = '';
    }
    isUploading = false;
  }

  async function handleLike(postId, isLiked) {
    const post = posts.find(p => p.id === postId);
    if (post) {
      const newLikes = (post.likes || 0) + (isLiked ? 1 : -1);
      await supabase.from('posts').update({ likes: newLikes }).eq('id', postId);
      post.likes = newLikes;
      posts = [...posts];
    }
  }

  // Burbujas pixeladas estilo retro
  const bubbles = [
    { left: '10%', delay: '0s', duration: '8s', size: '12px', color: '#A8B8A0' },
    { left: '20%', delay: '1s', duration: '5s', size: '8px', color: '#F4C2C2' },
    { left: '35%', delay: '2s', duration: '12s', size: '16px', color: '#A8C3D6' },
    { left: '50%', delay: '0.5s', duration: '7s', size: '10px', color: '#E8D5A0' },
    { left: '65%', delay: '3s', duration: '15s', size: '14px', color: '#C8A8D6' },
    { left: '80%', delay: '1.5s', duration: '9s', size: '9px', color: '#D6A8B8' },
    { left: '15%', delay: '4s', duration: '11s', size: '11px', color: '#A8B8A0' },
    { left: '45%', delay: '2.5s', duration: '6s', size: '7px', color: '#F4C2C2' },
    { left: '75%', delay: '5s', duration: '10s', size: '13px', color: '#A8C3D6' },
    { left: '90%', delay: '1s', duration: '14s', size: '15px', color: '#E8D5A0' }
  ];
</script>

<div class="app-wrapper">
  <!-- Burbujas pixeladas de fondo -->
  {#each bubbles as bubble}
    <div class="pixel-bubble" 
      style="
        --bubble-size: {bubble.size};
        --bubble-left: {bubble.left};
        --bubble-delay: {bubble.delay};
        --bubble-duration: {bubble.duration};
        --bubble-color: {bubble.color};
        --parallax-y: {scrollY * 0.01}px;
      "
    ></div>
  {/each}

  {#if showConfetti}<div class="confetti-container"></div>{/if}

  <!-- PANEL PRINCIPAL CON BORDE DOBLE -->
  <main class="content-panel">
    
    <!-- ENCABEZADO CON ESTILO RETRO -->
    <header class="panel-header">
      <div class="header-tab">
        <h1 class="logo">[ mecho ]</h1>
      </div>
      <p class="subtitle">tu diario de ánimo • pixel edition ✿</p>
      
      <!-- CONTROLES: Dark Mode Toggle -->
      <div class="header-controls">
        <button class="btn-theme-toggle" on:click={toggleDarkMode} title="Cambiar tema">
          {#if darkMode}☾{:else}☀{/if}
          <span class="theme-label">{darkMode ? 'dark' : 'light'}</span>
        </button>
      </div>
      
      <div class="stats-bar">
        <span class="stat-item">✿ {posts.length} entries</span>
        <span class="stat-separator">│</span>
        <span class="stat-item">★ {todayCount} today</span>
      </div>
    </header>

    <!-- SECCIÓN COMPOSITOR -->
    <section class="composer-panel">
      
      <!-- Preview de media -->
      {#if mediaPreview}
        <div class="preview-box">
          {#if mediaFile?.type?.startsWith('image')}
            <img src={mediaPreview} alt="Preview" class="preview-media" />
          {:else if mediaFile?.type?.startsWith('video')}
            <video src={mediaPreview} class="preview-media" muted />
          {:else if mediaFile?.type?.startsWith('audio')}
            <div class="preview-audio">♪ {mediaFile.name.slice(0, 18)}...</div>
          {/if}
          <button class="btn-remove" on:click={() => { 
            mediaPreview = null; mediaFile = null; 
            if (fileInputRef) fileInputRef.value = ''; 
          }}>✕</button>
        </div>
      {/if}

      <!-- Selector de Mood con estilo retro -->
      <div class="mood-selector">
        <div class="section-tab">
          <span class="tab-label">mood</span>
        </div>
        <div class="mood-options">
          {#each moods as mood}
            <button 
              class="mood-option {selectedMood === mood ? 'active' : ''}"
              style="border-color: {mood.border}; color: {mood.textColor}"
              on:click={() => selectMood(mood)}
              type="button"
            >
              <span class="mood-emoji">{mood.emoji}</span>
              <span class="mood-text">{mood.label}</span>
            </button>
          {/each}
        </div>
      </div>

      <!-- Textarea con borde definido -->
      <textarea 
        bind:value={textInput} 
        bind:this={textareaRef}
        placeholder="write your thoughts..."
        rows="3"
        class="composer-text"
      ></textarea>
      
      <!-- Botón emoji nativo -->
      <div class="emoji-helper">
        <button type="button" class="btn-emoji" on:click={openEmojiPicker}>
          ✿ emoji
          <span class="emoji-hint">[win+.]</span>
        </button>
      </div>
      
      <!-- Acciones del compositor -->
      <div class="composer-actions">
        <input 
          type="file" 
          accept="image/*,video/*,audio/*" 
          bind:this={fileInputRef} 
          on:change={handleFileChange} 
          class="file-input-hidden"
          id="media-input"
        />
        <label for="media-input" class="btn-attach">
          [ attach ]
        </label>
        <button 
          class="btn-send {isUploading ? 'uploading' : ''}" 
          on:click={sendPost} 
          disabled={(!textInput.trim() && !mediaFile) || isUploading || !selectedMood}
          type="button"
        >
          {#if isUploading}saving...{:else}save entry ✿{/if}
        </button>
      </div>
    </section>

    <!-- FEED DE POSTS CON PANEL DEFINIDO -->
    <section class="feed-panel">
      <div class="section-tab">
        <span class="tab-label">entries</span>
      </div>
      
      {#if loading}
        <div class="loader-box">
          <span class="loader-text">✿ loading mecho...</span>
        </div>
      {:else if posts.length === 0}
        <div class="empty-box">
          <span class="empty-icon">✦</span>
          <p>no entries yet</p>
          <small>start your first entry ✿</small>
        </div>
      {:else}
        {#each posts as post (post.id)}
          <div transition:fly={{ y: 15, duration: 300 }}>
            <Post {post} onLike={handleLike} {darkMode} />
          </div>
        {/each}
      {/if}
    </section>
    
  </main>

  <!-- MASCOTA MECHO - Pixel style -->
  <div class="mascot-container">
    <img src="/mecho.png" alt="Mecho" class="mascot-pixel" />
    <div class="mascot-bubble">
      {#if todayCount > 0}
        {todayCount} entry{todayCount > 1 ? 's' : ''} today ✿
      {:else}
        tell me about your day ✦
      {/if}
    </div>
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=VT323&family=Press+Start+2P&display=swap');

  /* === VARIABLES DE TEMA OSCURO (DEFAULT) === */
  :root {
    /* Fondos */
    --bg-main: #121212;
    --bg-panel: #1a1c1e;
    --bg-card: #1e2022;
    --bg-input: #242628;
    --bg-header: #2a2c2e;
    
    /* Texto */
    --text-main: #e8e6e3;
    --text-muted: #a8a6a3;
    --text-dark: #121212;
    
    /* Bordes pastel apagados */
    --border-mint: #A8B8A0;
    --border-rose: #F4C2C2;
    --border-ash: #A8C3D6;
    --border-lavender: #C8A8D6;
    --border-gold: #E8D5A0;
    --border-default: #3a3c3e;
    
    /* Acentos */
    --accent-pink: #F4C2C2;
    --accent-mint: #A8B8A0;
    --accent-blue: #A8C3D6;
    
    /* UI */
    --border-width: 2px;
    --border-dotted: 2px dotted;
    --border-dashed: 2px dashed;
    --radius: 4px;
    --radius-lg: 8px;
  }

  /* === VARIABLES DE TEMA CLARO === */
  body.light-mode {
    --bg-main: #f5f3f0;
    --bg-panel: #fffdf9;
    --bg-card: #faf8f5;
    --bg-input: #f0eee9;
    --bg-header: #e8e6e3;
    
    --text-main: #2a2c2e;
    --text-muted: #5a5c5e;
    --text-dark: #121212;
    
    --border-default: #c8c6c3;
  }

  /* === RESET Y BASE === */
  * { box-sizing: border-box; margin: 0; padding: 0; }
  
  body {
    background-color: var(--bg-main);
    background-image: 
      linear-gradient(rgba(168, 184, 160, 0.03) 1px, transparent 1px),
      linear-gradient(90deg, rgba(168, 184, 160, 0.03) 1px, transparent 1px);
    background-size: 20px 20px;
    color: var(--text-main);
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    line-height: 1.5;
    min-height: 100vh;
    overflow-x: hidden;
    image-rendering: pixelated;
  }

  .app-wrapper {
    position: relative;
    min-height: 100vh;
    padding: 20px;
    display: flex;
    justify-content: center;
  }

  /* === BURBUJAS PIXELADAS === */
  .pixel-bubble {
    position: fixed;
    bottom: -20px;
    left: var(--bubble-left);
    width: var(--bubble-size);
    height: var(--bubble-size);
    background: var(--bubble-color);
    border: 1px solid var(--text-muted);
    opacity: 0.6;
    pointer-events: none;
    z-index: 1;
    transform: translateY(var(--parallax-y));
    animation: floatPixel var(--bubble-duration) infinite steps(4);
    animation-delay: var(--bubble-delay);
    image-rendering: pixelated;
  }
  
  @keyframes floatPixel {
    0% { transform: translateY(0) translateX(0); opacity: 0; }
    10% { opacity: 0.6; }
    50% { transform: translateY(-40vh) translateX(20px); }
    100% { transform: translateY(-120vh) translateX(-10px); opacity: 0; }
  }

  .confetti-pixel {
    position: fixed;
    top: -10px;
    pointer-events: none;
    z-index: 9999;
    image-rendering: pixelated;
    animation: confettiPixel linear forwards;
  }
  @keyframes confettiPixel {
    0% { transform: translateY(0) rotate(0deg); opacity: 1; }
    100% { transform: translateY(100vh) rotate(360deg); opacity: 0; }
  }

  /* === PANEL PRINCIPAL CON BORDE DOBLE === */
  .content-panel {
    width: 100%;
    max-width: 640px;
    background: var(--bg-panel);
    border: var(--border-width) solid var(--border-mint);
    outline: var(--border-width) solid var(--border-ash);
    outline-offset: 4px;
    border-radius: var(--radius-lg);
    padding: 0;
    position: relative;
    z-index: 10;
  }

  /* === ENCABEZADO RETRO === */
  .panel-header {
    padding: 16px 20px;
    background: var(--bg-header);
    border-bottom: var(--border-width) solid var(--border-default);
    text-align: center;
    position: relative;
  }
  
  .header-tab {
    display: inline-block;
    background: var(--border-rose);
    color: var(--text-dark);
    padding: 4px 16px;
    border: var(--border-width) solid var(--text-dark);
    border-bottom: none;
    border-radius: var(--radius) var(--radius) 0 0;
    margin-bottom: -2px;
    position: relative;
    z-index: 2;
  }
  
  .logo {
    font-family: 'VT323', monospace;
    font-size: 1.8rem;
    letter-spacing: 2px;
    margin: 0;
    font-weight: normal;
  }
  
  .subtitle {
    color: var(--text-muted);
    font-size: 1rem;
    margin: 8px 0 12px;
    letter-spacing: 1px;
  }
  
  .header-controls {
    position: absolute;
    top: 12px;
    right: 16px;
  }
  
  .btn-theme-toggle {
    background: var(--bg-panel);
    border: var(--border-width) solid var(--border-ash);
    color: var(--text-main);
    padding: 4px 12px;
    font-family: 'VT323', monospace;
    font-size: 1rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 6px;
    transition: all 0.15s steps(2);
  }
  
  .btn-theme-toggle:hover {
    background: var(--border-ash);
    color: var(--text-dark);
  }
  
  .theme-label {
    font-size: 0.85rem;
    text-transform: lowercase;
  }
  
  .stats-bar {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 8px;
    padding-top: 8px;
    border-top: var(--border-dotted) var(--border-default);
    font-size: 0.95rem;
  }
  
  .stat-item { color: var(--text-muted); }
  .stat-separator { color: var(--border-default); }

  /* === SECCIONES CON TABS === */
  .section-tab {
    display: inline-block;
    background: var(--border-mint);
    color: var(--text-dark);
    padding: 3px 14px;
    border: var(--border-width) solid var(--text-dark);
    border-bottom: none;
    border-radius: var(--radius) var(--radius) 0 0;
    margin: -2px 0 12px 16px;
    position: relative;
    z-index: 2;
  }
  
  .tab-label {
    font-size: 0.9rem;
    text-transform: lowercase;
    letter-spacing: 1px;
    font-weight: normal;
  }

  /* === COMPOSITOR === */
  .composer-panel {
    padding: 20px;
    background: var(--bg-card);
    border-bottom: var(--border-width) solid var(--border-default);
  }

  .preview-box {
    position: relative;
    margin-bottom: 12px;
    border: var(--border-width) dashed var(--border-ash);
    border-radius: var(--radius);
    overflow: hidden;
  }
  
  .preview-media {
    width: 100%;
    max-height: 200px;
    object-fit: cover;
    display: block;
    image-rendering: pixelated;
  }
  
  .preview-audio {
    padding: 10px;
    text-align: center;
    color: var(--text-muted);
    font-size: 0.9rem;
  }
  
  .btn-remove {
    position: absolute;
    top: 4px;
    right: 4px;
    background: var(--border-rose);
    border: var(--border-width) solid var(--text-dark);
    color: var(--text-dark);
    width: 24px;
    height: 24px;
    border-radius: 2px;
    font-family: 'VT323', monospace;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.1s steps(2);
  }
  
  .btn-remove:hover {
    background: var(--text-dark);
    color: var(--border-rose);
  }

  /* === SELECTOR DE MOOD === */
  .mood-selector { margin-bottom: 16px; }
  
  .mood-options {
    display: flex;
    gap: 6px;
    flex-wrap: wrap;
  }
  
  .mood-option {
    background: var(--bg-input);
    border: var(--border-width) solid;
    border-radius: var(--radius);
    padding: 6px 12px;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 0.9rem;
    transition: all 0.1s steps(2);
    font-family: 'VT323', monospace;
  }
  
  .mood-option:hover {
    background: var(--bg-panel);
  }
  
  .mood-option.active {
    background: var(--bg-header);
    box-shadow: inset 0 0 0 2px var(--text-dark);
  }
  
  .mood-emoji { font-size: 1.1rem; }
  .mood-text { text-transform: lowercase; }

  /* === TEXTAREA === */
  .composer-text {
    width: 100%;
    border: var(--border-width) solid var(--border-default);
    background: var(--bg-input);
    color: var(--text-main);
    border-radius: var(--radius);
    padding: 12px;
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    resize: vertical;
    min-height: 80px;
    transition: border-color 0.1s steps(2);
  }
  
  .composer-text:focus {
    outline: none;
    border-color: var(--border-mint);
    box-shadow: inset 0 0 0 2px var(--border-mint);
  }
  
  .composer-text::placeholder {
    color: var(--text-muted);
    opacity: 0.7;
  }

  /* === BOTÓN EMOJI === */
  .emoji-helper {
    margin: 10px 0 16px;
    text-align: center;
  }
  
  .btn-emoji {
    background: var(--bg-input);
    border: var(--border-width) solid var(--border-lavender);
    color: var(--text-main);
    padding: 5px 14px;
    border-radius: var(--radius);
    font-family: 'VT323', monospace;
    font-size: 0.95rem;
    cursor: pointer;
    transition: all 0.1s steps(2);
  }
  
  .btn-emoji:hover {
    background: var(--border-lavender);
    color: var(--text-dark);
  }
  
  .emoji-hint {
    color: var(--text-muted);
    font-size: 0.8rem;
    margin-left: 4px;
    opacity: 0.8;
  }

  /* === ACCIONES === */
  .composer-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
  }

  .file-input-hidden { display: none; }
  
  .btn-attach {
    background: var(--bg-input);
    border: var(--border-width) solid var(--border-ash);
    color: var(--text-main);
    padding: 8px 16px;
    border-radius: var(--radius);
    font-family: 'VT323', monospace;
    font-size: 0.95rem;
    cursor: pointer;
    transition: all 0.1s steps(2);
    text-transform: lowercase;
  }
  
  .btn-attach:hover {
    background: var(--border-ash);
    color: var(--text-dark);
  }

  .btn-send {
    background: var(--border-mint);
    border: var(--border-width) solid var(--text-dark);
    color: var(--text-dark);
    padding: 8px 24px;
    border-radius: var(--radius);
    font-family: 'VT323', monospace;
    font-size: 1rem;
    font-weight: normal;
    cursor: pointer;
    transition: all 0.1s steps(2);
    text-transform: lowercase;
  }
  
  .btn-send:hover:not(:disabled) {
    background: var(--text-dark);
    color: var(--border-mint);
  }
  
  .btn-send:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .btn-send.uploading {
    animation: blink 0.5s steps(2) infinite;
  }
  
  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.6; }
  }

  /* === FEED PANEL === */
  .feed-panel {
    padding: 20px;
    background: var(--bg-card);
  }

  .loader-box, .empty-box {
    text-align: center;
    padding: 32px 20px;
    border: var(--border-dashed) var(--border-default);
    border-radius: var(--radius);
    color: var(--text-muted);
  }
  
  .loader-text { font-size: 1.1rem; }
  .empty-icon { 
    font-size: 2rem; 
    display: block; 
    margin-bottom: 8px;
    color: var(--border-rose);
  }
  .empty-box small { 
    display: block; 
    margin-top: 4px; 
    color: var(--text-muted);
    font-size: 0.9rem;
  }

  /* === MASCOTA PIXEL === */
  .mascot-container {
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 20;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 8px;
  }
  
  .mascot-pixel {
    width: 60px;
    height: auto;
    image-rendering: pixelated;
    animation: floatPixelMascot 2s infinite steps(4);
    filter: drop-shadow(2px 2px 0 var(--border-default));
  }
  
  @keyframes floatPixelMascot {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-8px); }
  }
  
  .mascot-bubble {
    background: var(--bg-panel);
    border: var(--border-width) solid var(--border-rose);
    border-radius: var(--radius);
    padding: 6px 12px;
    font-size: 0.9rem;
    color: var(--text-main);
    max-width: 180px;
    text-align: right;
    position: relative;
  }
  
  .mascot-bubble::before {
    content: '';
    position: absolute;
    bottom: -8px;
    right: 12px;
    border: 8px solid transparent;
    border-top-color: var(--border-rose);
    border-bottom: 0;
  }

  /* === CONFETTI CONTAINER === */
  .confetti-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 9998;
  }

  /* === RESPONSIVE === */
  @media (max-width: 680px) {
    .app-wrapper { padding: 12px; }
    .content-panel { 
      outline-offset: 2px; 
      border-radius: var(--radius);
    }
    .panel-header { padding: 12px 16px; }
    .logo { font-size: 1.5rem; }
    .composer-panel, .feed-panel { padding: 16px; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .mascot-container { 
      bottom: 12px; 
      right: 12px; 
    }
    .mascot-pixel { width: 48px; }
    .pixel-bubble { --bubble-size: calc(var(--bubble-size) * 0.7) !important; }
  }

  @media (max-width: 400px) {
    .pixel-bubble { display: none; }
    .emoji-hint { display: none; }
    .mood-options { justify-content: center; }
    .header-controls { 
      position: static; 
      margin-top: 8px; 
      display: flex; 
      justify-content: center;
    }
  }
</style>