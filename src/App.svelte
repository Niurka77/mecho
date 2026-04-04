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
  let textareaRef; // ← Nuevo: referencia al textarea
  let scrollY = 0;
  let channel;
  
  let selectedColor = '#FFE5E5';
  let selectedMood = null;
  let isUploading = false;
  let showConfetti = false;
  let todayCount = 0;
  let popSound;
  
  // MOODS DISPONIBLES
  const moods = [
    { emoji: '🌸', label: 'Feliz', color: '#FFE5E5', textColor: '#8B4A4A' },
    { emoji: '🌧️', label: 'Tranquilo', color: '#E5F3FF', textColor: '#4A6B8B' },
    { emoji: '⚡', label: 'Energético', color: '#FFF9E5', textColor: '#8B7D4A' },
    { emoji: '🍃', label: 'En paz', color: '#E5FFE5', textColor: '#4A8B4A' },
    { emoji: '💜', label: 'Melancólico', color: '#F3E5FF', textColor: '#6B4A8B' },
    { emoji: '🔥', label: 'Motivado', color: '#FFE5F3', textColor: '#8B4A6B' }
  ];

  onMount(async () => {
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

  // ← NUEVO: Enfocar textarea para activar picker nativo de emojis
  function openEmojiPicker() {
    if (textareaRef) {
      textareaRef.focus();
      // En móviles, esto suele abrir el teclado con acceso a emojis
      // En desktop, el usuario puede usar Win+. o Ctrl+Cmd+Espacio
    }
  }

  function createConfetti() {
    showConfetti = true;
    const colors = moods.map(m => m.color);
    for (let i = 0; i < 60; i++) {
      const confetti = document.createElement('div');
      confetti.className = 'confetti';
      confetti.style.left = Math.random() * 100 + '%';
      confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
      confetti.style.width = Math.random() * 8 + 4 + 'px';
      confetti.style.height = Math.random() * 8 + 4 + 'px';
      confetti.style.animationDuration = Math.random() * 2 + 1 + 's';
      document.body.appendChild(confetti);
      setTimeout(() => confetti.remove(), 3000);
    }
    setTimeout(() => { showConfetti = false; }, 3000);
  }

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✨ Escribe algo bonito');
      return;
    }
    if (!selectedMood) {
      alert('🌸 Selecciona cómo te sientes');
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
        alert('No se pudo subir');
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
      type: postType,
      likes: 0
    };

    const { error: dbError } = await supabase.from('posts').insert([newPostData]);
    if (dbError) {
      console.error('Error BD:', dbError);
      alert('Algo salió mal');
    } else {
      createConfetti();
      if (popSound) popSound.play();
      textInput = ''; 
      mediaFile = null; 
      mediaPreview = null;
      selectedMood = null;
      selectedColor = '#FFE5E5';
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

  // Burbujas con colores más definidos para mejor visibilidad
  const bubbles = [
    { left: '10%', delay: '0s', duration: '8s', size: '80px', color: '#ffb3c1' },
    { left: '20%', delay: '1s', duration: '5s', size: '40px', color: '#ffd9b3' },
    { left: '35%', delay: '2s', duration: '12s', size: '100px', color: '#d4c1ec' },
    { left: '50%', delay: '0.5s', duration: '7s', size: '60px', color: '#f9c1d9' },
    { left: '65%', delay: '3s', duration: '15s', size: '120px', color: '#b3e0f2' },
    { left: '80%', delay: '1.5s', duration: '9s', size: '50px', color: '#d9d9e6' },
    { left: '15%', delay: '4s', duration: '11s', size: '90px', color: '#c1d4f2' },
    { left: '45%', delay: '2.5s', duration: '6s', size: '30px', color: '#ffe6cc' },
    { left: '75%', delay: '5s', duration: '10s', size: '70px', color: '#e6f9b3' },
    { left: '90%', delay: '1s', duration: '14s', size: '110px', color: '#c1f2d4' }
  ];
</script>

<div class="app-wrapper">
  {#each bubbles as bubble, i}
    <div class="bubble" 
      style="
        --bubble-size: {bubble.size};
        --bubble-left: {bubble.left};
        --bubble-delay: {bubble.delay};
        --bubble-duration: {bubble.duration};
        --bubble-color: {bubble.color};
        --parallax-y: {scrollY * 0.02 * (i + 1)}px;
      "
    ></div>
  {/each}

  {#if showConfetti}<div class="confetti-container"></div>{/if}

  <main class="content">
    <header>
      <h1 class="logo">mecho</h1>
      <p class="subtitle">tu diario de ánimo suave 🌸</p>
      <div class="stats">
        <span class="stat-badge">📝 {posts.length} días registrados</span>
        <span class="stat-badge">💫 {todayCount} hoy</span>
      </div>
    </header>

    <section class="composer">
      {#if mediaPreview}
        <div class="preview-wrapper">
          {#if mediaFile?.type?.startsWith('image')}
            <img src={mediaPreview} alt="Preview" class="preview-media" />
          {:else if mediaFile?.type?.startsWith('video')}
            <video src={mediaPreview} class="preview-media" muted />
          {:else if mediaFile?.type?.startsWith('audio')}
            <div class="preview-audio">🎵 {mediaFile.name.slice(0, 20)}...</div>
          {/if}
          <button class="remove-preview" on:click={() => { 
            mediaPreview = null; mediaFile = null; 
            if (fileInputRef) fileInputRef.value = ''; 
          }}>✕</button>
        </div>
      {/if}

      <!-- SELECTOR DE MOOD -->
      <div class="mood-selector">
        <p class="mood-label">¿Cómo te sientes hoy?</p>
        <div class="mood-options">
          {#each moods as mood}
            <button 
              class="mood-option {selectedMood === mood ? 'active' : ''}"
              style="background-color: {mood.color}; color: {mood.textColor}"
              on:click={() => selectMood(mood)}
              type="button"
            >
              <span class="mood-emoji">{mood.emoji}</span>
              <span class="mood-text">{mood.label}</span>
            </button>
          {/each}
        </div>
      </div>

      <textarea 
        bind:value={textInput} 
        bind:this={textareaRef}
        placeholder="Cuéntame sobre tu día..."
        rows="3"
        class="composer-text"
      ></textarea>
      
      <!-- BOTÓN PARA EMOJIS NATIVOS -->
      <div class="emoji-helper">
        <button type="button" class="btn-emoji-picker" on:click={openEmojiPicker} title="Insertar emoji">
          🪄 Emojis
          <span class="emoji-hint">Win+. o Ctrl+Cmd+Espacio</span>
        </button>
      </div>
      
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
          📎 {mediaFile ? mediaFile.name.slice(0, 18) : 'adjuntar'}
        </label>
        <button 
          class="btn-send {isUploading ? 'uploading' : ''}" 
          on:click={sendPost} 
          disabled={(!textInput.trim() && !mediaFile) || isUploading || !selectedMood}
          type="button"
        >
          {#if isUploading}guardando...{:else}guardar ✨{/if}
        </button>
      </div>
    </section>

    <section class="feed">
      {#if loading}
        <div class="loader">🌸 mecho está despertando...</div>
      {:else if posts.length === 0}
        <div class="empty-state">
          <span class="empty-emoji">🫧</span>
          <p>Aún no hay registros</p>
          <small>¡Empieza hoy! 💫</small>
        </div>
      {:else}
        {#each posts as post (post.id)}
          <div transition:fly={{ y: 20, duration: 500 }}>
            <Post {post} onLike={handleLike} />
          </div>
        {/each}
      {/if}
    </section>
  </main>

  <div class="mascot-wrapper">
    <img src="/mecho.png" alt="Mecho" class="mascot" />
    <div class="mascot-tooltip">
      {#if todayCount > 0}
        ¡{todayCount} registro{todayCount > 1 ? 's' : ''} hoy! 💖
      {:else}
        ¡Cuéntame tu día! 🌸
      {/if}
    </div>
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;500;600;700;800&family=VT323&display=swap');

  :root {
    --bg: #FAF7F2;
    --bg-gradient: linear-gradient(135deg, #FAF7F2 0%, #F5F0E8 100%);
    --text: #3D3D3D;
    --text-light: #6B6B6B;
    --text-dark: #2D2D2D;
    --card: #FFFFFF;
    --green: #7A8A6C;
    --green-soft: #9AAF92;
    --blue: #9DB5C7;
    --pink: #D4A5A5;
    --sand: #E8D5C4;
    --shadow: 0 8px 28px rgba(61, 61, 61, 0.08);
    --shadow-hover: 0 14px 35px rgba(61, 61, 61, 0.12);
    --radius-lg: 32px;
    --radius-md: 24px;
    --radius-sm: 18px;
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }
  
  body {
    background: var(--bg);
    background-image: var(--bg-gradient);
    color: var(--text);
    font-family: 'Nunito', sans-serif;
    scroll-behavior: smooth;
    min-height: 100vh;
    overflow-x: hidden;
  }

  .app-wrapper {
    position: relative;
    min-height: 100vh;
    padding-bottom: 80px;
    overflow: hidden;
  }

  /* === BURBUJAS MEJORADAS - MÁS VISIBLES === */
  .bubble {
    position: fixed;
    bottom: -150px;
    left: var(--bubble-left);
    width: var(--bubble-size);
    height: var(--bubble-size);
    background: rgba(255, 255, 255, 0.85); /* ← Más opaco */
    border: 2px solid rgba(200, 220, 255, 0.6); /* ← Borde más definido */
    border-radius: 50%;
    box-shadow: 
      0 8px 25px rgba(61, 61, 61, 0.12),
      inset 0 10px 25px 5px rgba(255, 255, 255, 0.98),
      inset 0 -12px 25px rgba(180, 200, 240, 0.4),
      inset 0 0 25px var(--bubble-color); /* ← Brillo de color más intenso */
    backdrop-filter: blur(2px);
    pointer-events: none;
    z-index: 1;
    transform: translateY(var(--parallax-y));
    animation: floatUp var(--bubble-duration) infinite ease-in;
    animation-delay: var(--bubble-delay);
    opacity: 0.9; /* ← Opacidad aumentada */
  }
  
  .bubble::after {
    content: "";
    position: absolute;
    top: 12%; left: 12%;
    width: 30%; height: 22%;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.98);
    transform: rotate(-30deg);
    filter: blur(0.3px);
    box-shadow: 0 2px 6px rgba(255,255,255,0.5);
  }
  
  @keyframes floatUp {
    0% { transform: translateY(0) translateX(0); opacity: 0; }
    10% { opacity: 0.9; }
    50% { transform: translateY(-50vh) translateX(30px); }
    100% { transform: translateY(-120vh) translateX(-15px); opacity: 0; }
  }

  .confetti {
    position: fixed;
    top: -10px;
    pointer-events: none;
    z-index: 9999;
    border-radius: 2px;
    animation: confettiFall linear forwards;
  }
  @keyframes confettiFall {
    0% { transform: translateY(0) rotate(0deg); opacity: 1; }
    100% { transform: translateY(100vh) rotate(720deg); opacity: 0; }
  }

  .content {
    max-width: 660px;
    margin: 0 auto;
    padding: 24px 20px 100px;
    position: relative;
    z-index: 10;
  }

  header {
    text-align: center;
    margin-bottom: 28px;
    padding-top: 12px;
  }
  
  .logo {
    font-family: 'VT323', monospace;
    font-size: 3rem;
    color: var(--green);
    letter-spacing: 4px;
    text-shadow: 0 2px 6px rgba(122, 138, 108, 0.15);
  }
  
  .subtitle {
    color: var(--text-light);
    font-size: 1rem;
    margin-top: 6px;
    font-weight: 500;
  }
  
  .stats {
    display: flex;
    justify-content: center;
    gap: 16px;
    margin-top: 14px;
    flex-wrap: wrap;
  }
  
  .stat-badge {
    background: rgba(157, 181, 199, 0.25);
    padding: 6px 16px;
    border-radius: 40px;
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--text);
    backdrop-filter: blur(4px);
    border: 1px solid rgba(255, 255, 255, 0.6);
  }

  .mood-selector { margin-bottom: 16px; }
  
  .mood-label {
    font-size: 0.9rem;
    color: var(--text-light);
    margin-bottom: 10px;
    font-weight: 600;
    text-align: center;
  }
  
  .mood-options {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    justify-content: center;
    margin-bottom: 12px;
  }
  
  .mood-option {
    padding: 8px 16px;
    border-radius: 40px;
    border: 2px solid transparent;
    cursor: pointer;
    transition: all 0.25s cubic-bezier(0.34, 1.2, 0.64, 1);
    display: flex;
    align-items: center;
    gap: 6px;
    font-weight: 600;
    font-size: 0.9rem;
    box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  }
  
  .mood-option:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.12);
  }
  
  .mood-option.active {
    border-color: var(--green);
    transform: scale(1.05);
    box-shadow: 0 0 0 3px rgba(122, 138, 108, 0.25);
  }
  
  .mood-emoji { font-size: 1.2rem; }

  .composer {
    background: rgba(255, 255, 255, 0.95);
    backdrop-filter: blur(12px);
    border-radius: var(--radius-lg);
    padding: 24px;
    box-shadow: var(--shadow);
    margin-bottom: 32px;
    border: 2px solid rgba(255, 255, 255, 0.9);
    transition: all 0.3s ease;
  }
  
  .composer:hover {
    box-shadow: var(--shadow-hover);
    transform: translateY(-2px);
  }

  .preview-wrapper { position: relative; margin-bottom: 12px; }
  
  .preview-media {
    width: 100%;
    max-height: 220px;
    border-radius: var(--radius-md);
    object-fit: cover;
    box-shadow: 0 8px 24px rgba(0,0,0,0.08);
    border: 3px solid white;
  }
  
  .preview-audio {
    background: var(--sand);
    padding: 12px 16px;
    border-radius: var(--radius-md);
    font-size: 0.9rem;
    color: var(--text);
    text-align: center;
    font-weight: 500;
  }
  
  .remove-preview {
    position: absolute;
    top: 8px; right: 8px;
    background: rgba(255, 255, 255, 0.98);
    border: none;
    border-radius: 50%;
    width: 32px; height: 32px;
    font-size: 1.2rem;
    color: var(--text);
    cursor: pointer;
    transition: all 0.2s;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  }
  
  .remove-preview:hover { 
    transform: scale(1.1) rotate(90deg);
    background: var(--pink);
  }

  .composer-text {
    width: 100%;
    border: none;
    background: var(--sand);
    border-radius: var(--radius-md);
    padding: 14px 18px;
    font-family: 'Nunito', sans-serif;
    font-size: 1rem;
    resize: vertical;
    color: var(--text-dark);
    line-height: 1.6;
    transition: all 0.2s;
    font-weight: 500;
  }
  
  .composer-text:focus {
    outline: 3px solid var(--pink);
    outline-offset: 3px;
    background: white;
  }

  /* === BOTÓN DE EMOJIS NATIVOS === */
  .emoji-helper {
    display: flex;
    justify-content: center;
    margin: 8px 0 16px;
  }
  
  .btn-emoji-picker {
    background: linear-gradient(135deg, rgba(232,213,196,0.6), rgba(244,228,210,0.8));
    border: 1px solid rgba(212,165,165,0.4);
    font-size: 0.9rem;
    padding: 8px 20px;
    border-radius: 40px;
    cursor: pointer;
    transition: all 0.25s ease;
    color: var(--text);
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 8px;
    position: relative;
  }
  
  .btn-emoji-picker:hover {
    transform: translateY(-2px);
    background: linear-gradient(135deg, rgba(232,213,196,0.8), rgba(244,228,210,1));
    box-shadow: 0 4px 12px rgba(212,165,165,0.25);
  }
  
  .emoji-hint {
    font-size: 0.75rem;
    color: var(--text-light);
    font-weight: 400;
    opacity: 0.85;
    margin-left: 4px;
  }

  .composer-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    margin-top: 8px;
  }

  .file-input-hidden { display: none; }
  
  .btn-attach {
    background: var(--blue);
    color: white;
    padding: 10px 20px;
    border-radius: 30px;
    font-weight: 600;
    font-size: 0.9rem;
    cursor: pointer;
    transition: all 0.2s;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 200px;
  }
  
  .btn-attach:hover { 
    background: #8AA3B8; 
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(157, 181, 199, 0.45);
  }

  .btn-send {
    background: var(--green);
    color: white;
    border: none;
    padding: 12px 32px;
    border-radius: 32px;
    font-family: 'Nunito', sans-serif;
    font-weight: 700;
    font-size: 1rem;
    cursor: pointer;
    transition: all 0.2s;
    box-shadow: 0 4px 12px rgba(122, 138, 108, 0.3);
  }
  
  .btn-send:hover:not(:disabled) { 
    transform: translateY(-2px) scale(1.02); 
    background: #6A7A5C;
    box-shadow: 0 8px 20px rgba(122, 138, 108, 0.45);
  }
  
  .btn-send:active:not(:disabled) { transform: scale(0.98); }
  .btn-send:disabled { opacity: 0.5; cursor: not-allowed; }
  .btn-send.uploading {
    background: var(--sand);
    animation: pulse 1.5s ease-in-out infinite;
    color: var(--text);
  }
  
  @keyframes pulse { 0%, 100% { opacity: 0.75; } 50% { opacity: 1; } }

  .feed { display: flex; flex-direction: column; gap: 20px; }
  
  .loader, .empty-state {
    text-align: center;
    padding: 40px 24px;
    background: rgba(255,255,255,0.9);
    backdrop-filter: blur(8px);
    border-radius: var(--radius-lg);
    border: 2px dashed var(--green-soft);
    color: var(--text);
  }
  
  .loader { font-weight: 600; }
  .empty-emoji { font-size: 3rem; display: block; margin-bottom: 12px; }

  .mascot-wrapper {
    position: fixed;
    bottom: 24px;
    right: 20px;
    z-index: 20;
    cursor: pointer;
  }
  
  .mascot {
    width: 70px;
    height: auto;
    animation: floatMascot 3s ease-in-out infinite;
    filter: drop-shadow(0 6px 12px rgba(61,61,61,0.15));
    transition: transform 0.2s;
  }
  
  .mascot:hover { transform: scale(1.05); }
  .mascot:hover + .mascot-tooltip { opacity: 1; transform: translateY(0); }
  
  .mascot-tooltip {
    position: absolute;
    bottom: 75px;
    right: 0;
    background: white;
    padding: 8px 16px;
    border-radius: 30px;
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--text);
    white-space: nowrap;
    box-shadow: 0 4px 12px rgba(0,0,0,0.12);
    opacity: 0;
    transform: translateY(10px);
    transition: all 0.3s ease;
    pointer-events: none;
    border: 1px solid var(--sand);
  }
  
  @keyframes floatMascot {
    0%, 100% { transform: translateY(0) rotate(-3deg); }
    50% { transform: translateY(-12px) rotate(3deg); }
  }

  @media (max-width: 768px) {
    .bubble { --bubble-size: calc(var(--bubble-size) * 0.75) !important; }
    .mood-options { gap: 6px; }
    .mood-option { padding: 6px 12px; font-size: 0.85rem; }
  }

  @media (max-width: 560px) {
    .logo { font-size: 2.5rem; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .mascot { width: 55px; }
    .stats { flex-direction: column; align-items: center; gap: 8px; }
    .composer { padding: 18px; }
    .bubble { --bubble-size: calc(var(--bubble-size) * 0.55) !important; }
    .mood-options { flex-direction: column; }
    .mood-option { justify-content: center; }
    .emoji-hint { display: none; } /* Ocultar hint en móviles para ahorrar espacio */
  }

  @media (max-width: 380px) {
    .bubble { display: none; }
  }
</style>