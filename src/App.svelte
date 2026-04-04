<script>
  import { onMount, onDestroy } from 'svelte';
  import { fade, fly } from 'svelte/transition';
  import { supabase } from './lib/supabaseClient';
  import Post from './lib/Post.svelte';

  // --- ESTADO ---
  let posts = [];
  let loading = true;
  let textInput = '';
  let mediaFile = null;
  let mediaPreview = null;
  let fileInputRef;
  let scrollY = 0;
  let channel;
  
  // Estados para colores y upload
  let selectedColor = '#FFFFFF';
  let isUploading = false;
  let showConfetti = false;
  let todayCount = 0;
  
  // Sonido
  let popSound;
  
  // Colores pastel disponibles
  const pastelColors = [
    { value: '#FFE5E5', label: 'Rosa pastel' },
    { value: '#E5F3FF', label: 'Azul pastel' },
    { value: '#E5FFE5', label: 'Verde pastel' },
    { value: '#FFF9E5', label: 'Amarillo pastel' },
    { value: '#FDE2F3', label: 'Lavanda' },
    { value: '#E0F7FA', label: 'Menta' }
  ];

  // Emojis rápidos
  const quickEmojis = ['🌸', '✨', '💕', '🫧', '⭐', '🌙', '🍰', '🐱', '💖', '🎀'];

  // --- CARGAR POSTS + TIEMPO REAL ---
  onMount(async () => {
    // Cargar sonido
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    // Cargar posts
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) console.error('Error cargando posts:', error);
    else posts = data || [];
    loading = false;

    // Contar posts de hoy
    updateTodayCount();

    // Suscribirse a nuevos posts
    channel = supabase
      .channel('mecho-realtime')
      .on('postgres_changes', 
        { event: 'INSERT', schema: 'public', table: 'posts' }, 
        payload => {
          posts = [payload.new, ...posts];
          updateTodayCount();
          // Reproducir sonido suave
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

  function handleScroll() {
    scrollY = window.scrollY;
  }
  
  function updateTodayCount() {
    const today = new Date().toDateString();
    todayCount = posts.filter(p => new Date(p.created_at).toDateString() === today).length;
  }

  // --- PREVISUALIZACIÓN DE ARCHIVO ---
  function handleFileChange(event) {
    mediaFile = event.target.files[0];
    if (mediaFile) {
      const reader = new FileReader();
      reader.onload = e => { mediaPreview = e.target.result; };
      reader.readAsDataURL(mediaFile);
    } else {
      mediaPreview = null;
    }
  }

  // --- CONFETTI ---
  function createConfetti() {
    showConfetti = true;
    const colors = ['#FFE5E5', '#E5F3FF', '#E5FFE5', '#FFF9E5', '#F4C2C2', '#FDE2F3'];
    for (let i = 0; i < 60; i++) {
      const confetti = document.createElement('div');
      confetti.className = 'confetti';
      confetti.style.left = Math.random() * 100 + '%';
      confetti.style.backgroundColor = colors[Math.floor(Math.random() * colors.length)];
      confetti.style.width = Math.random() * 8 + 4 + 'px';
      confetti.style.height = Math.random() * 8 + 4 + 'px';
      confetti.style.animationDuration = Math.random() * 2 + 1 + 's';
      confetti.style.animationDelay = Math.random() * 0.5 + 's';
      document.body.appendChild(confetti);
      setTimeout(() => confetti.remove(), 3000);
    }
    setTimeout(() => { showConfetti = false; }, 3000);
  }

  // --- ENVIAR POST ---
  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✨ Escribe algo o adjunta un recuerdo');
      return;
    }

    isUploading = true;
    let currentMediaUrl = null;

    // Subir archivo si existe
    if (mediaFile) {
      const fileExt = mediaFile.name.split('.').pop();
      const safeName = `${Date.now()}_${Math.random().toString(36).slice(2, 8)}.${fileExt}`;
      const datePath = new Date().toISOString().split('T')[0];
      const filePath = `posts/${datePath}/${safeName}`;

      const { error: uploadError } = await supabase.storage
        .from('mecho-media')
        .upload(filePath, mediaFile, { cacheControl: '3600' });

      if (uploadError) {
        console.error('Error subiendo:', uploadError);
        alert('No se pudo subir el archivo. Intenta de nuevo.');
        isUploading = false;
        return;
      }
      
      const { data: { publicUrl } } = supabase.storage
        .from('mecho-media')
        .getPublicUrl(filePath);
      currentMediaUrl = publicUrl;
    }

    // Determinar tipo
    let postType = 'note';
    if (mediaFile) {
      if (mediaFile.type.startsWith('image')) postType = 'image';
      else if (mediaFile.type.startsWith('video')) postType = 'video';
      else if (mediaFile.type.startsWith('audio')) postType = 'audio';
    }

    // Crear post
    const newPostData = {
      text: textInput.trim() || null,
      media_url: currentMediaUrl,
      color: selectedColor,
      type: postType,
      likes: 0
    };

    const { error: dbError } = await supabase
      .from('posts')
      .insert([newPostData]);

    if (dbError) {
      console.error('Error en BD:', dbError);
      alert('Algo salió mal. Revisa tu conexión.');
    } else {
      // ¡Éxito! Animaciones
      createConfetti();
      if (popSound) popSound.play();
      
      // Limpiar formulario
      textInput = '';
      mediaFile = null;
      mediaPreview = null;
      selectedColor = '#FFFFFF';
      if (fileInputRef) fileInputRef.value = '';
    }
    
    isUploading = false;
  }

  // Agregar emoji al texto
  function addEmoji(emoji) {
    textInput += emoji;
  }

  // Manejar like desde Post component
  async function handleLike(postId, isLiked) {
    const post = posts.find(p => p.id === postId);
    if (post) {
      const newLikes = (post.likes || 0) + (isLiked ? 1 : -1);
      await supabase
        .from('posts')
        .update({ likes: newLikes })
        .eq('id', postId);
      post.likes = newLikes;
      posts = [...posts];
    }
  }

  // --- BURBUJAS FLOTANTES (más burbujas) ---
  const bubbles = [
    { top: '15%', left: '5%', size: 55, delay: 0, duration: 20 },
    { top: '70%', right: '3%', size: 45, delay: 2, duration: 24 },
    { bottom: '15%', left: '8%', size: 38, delay: 4, duration: 18 },
    { top: '30%', right: '12%', size: 32, delay: 1, duration: 22 },
    { bottom: '45%', right: '18%', size: 48, delay: 3, duration: 26 },
    { top: '85%', left: '15%', size: 28, delay: 5, duration: 19 },
    { top: '8%', right: '20%', size: 42, delay: 2.5, duration: 23 },
    { bottom: '60%', left: '20%', size: 35, delay: 1.5, duration: 21 },
    { top: '50%', left: '2%', size: 30, delay: 3.5, duration: 25 },
    { bottom: '10%', right: '25%', size: 40, delay: 4.5, duration: 20 }
  ];
</script>

<div class="app-wrapper">
  
  <!-- Burbujas flotantes -->
  {#each bubbles as bubble, i}
    <div 
      class="floating-bubble" 
      style="
        --size: {bubble.size}px;
        --top: {bubble.top};
        --left: {bubble.left};
        --right: {bubble.right};
        --bottom: {bubble.bottom};
        --delay: {bubble.delay}s;
        --duration: {bubble.duration}s;
        --parallax: {scrollY * 0.03 * (i + 1)}px;
      "
    >
      <span class="bubble-content">🫧</span>
    </div>
  {/each}

  <!-- Confetti -->
  {#if showConfetti}
    <div class="confetti-container"></div>
  {/if}

  <main class="content">
    <header>
      <h1 class="logo">mecho ✨</h1>
      <p class="subtitle">un rincón suave para nosotras</p>
      <div class="stats">
        <span class="stat-badge">🌸 {posts.length} recuerdos</span>
        <span class="stat-badge">💫 {todayCount} hoy</span>
      </div>
    </header>

    <!-- Caja para crear posts -->
    <section class="composer">
      {#if mediaPreview}
        <div class="preview-wrapper">
          {#if mediaFile?.type?.startsWith('image')}
            <img src={mediaPreview} alt="Previsualización" class="preview-media" />
          {:else if mediaFile?.type?.startsWith('video')}
            <video src={mediaPreview} class="preview-media" muted />
          {:else if mediaFile?.type?.startsWith('audio')}
            <div class="preview-audio">🎵 Audio: {mediaFile.name.slice(0, 20)}...</div>
          {/if}
          <button class="remove-preview" on:click={() => { 
            mediaPreview = null; mediaFile = null; 
            if (fileInputRef) fileInputRef.value = ''; 
          }}>✕</button>
        </div>
      {/if}

      <textarea 
        bind:value={textInput} 
        placeholder="Escribe un mensajito bonito... ✏️"
        rows="3"
        class="composer-text"
      ></textarea>
      
      <!-- Emojis rápidos -->
      <div class="quick-emojis">
        {#each quickEmojis as emoji}
          <button class="emoji-btn" on:click={() => addEmoji(emoji)}>
            {emoji}
          </button>
        {/each}
      </div>
      
      <!-- Selector de colores -->
      <div class="color-selector">
        <span class="color-label">🎨 colorcito:</span>
        <div class="color-options">
          {#each pastelColors as colorOption}
            <button
              class="color-option {selectedColor === colorOption.value ? 'active' : ''}"
              style="background-color: {colorOption.value}"
              on:click={() => selectedColor = colorOption.value}
              title={colorOption.label}
            >
              {#if selectedColor === colorOption.value}
                <span class="checkmark">✓</span>
              {/if}
            </button>
          {/each}
        </div>
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
          📎 {mediaFile ? mediaFile.name.slice(0, 18) + (mediaFile.name.length > 18 ? '...' : '') : 'adjuntar'}
        </label>
        <button 
          class="btn-send {isUploading ? 'uploading' : ''}" 
          on:click={sendPost} 
          disabled={!textInput.trim() && !mediaFile || isUploading}
        >
          {#if isUploading}
            🐾 mecho está trabajando...
          {:else}
            enviar ✨
          {/if}
        </button>
      </div>
    </section>

    <!-- Feed de posts -->
    <section class="feed">
      {#if loading}
        <div class="loader">
          <span class="loader-emoji">🌸</span>
          mecho está despertando...
        </div>
      {:else if posts.length === 0}
        <div class="empty-state">
          <span class="empty-emoji">🫧</span>
          <p>Aún no hay recuerdos</p>
          <small>¡Sé la primera en compartir algo bonito! 💫</small>
        </div>
      {:else}
        {#each posts as post (post.id)}
          <div transition:fly={{ y: 20, duration: 500, opacity: 0 }}>
            <Post {post} onLike={handleLike} />
          </div>
        {/each}
      {/if}
    </section>
  </main>

  <!-- Mascota flotante -->
  <div class="mascot-wrapper">
    <img src="/mecho.png" alt="Mecho" class="mascot" />
    <div class="mascot-tooltip">¡hola! 💖</div>
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;500;600;700;800&family=VT323&display=swap');

  :root {
    --bg: #FDFBF5;
    --bg-gradient: linear-gradient(135deg, #FDFBF5 0%, #FFF9F0 50%, #FDFBF5 100%);
    --card: #FFFFFF;
    --green: #8B9A7C;
    --green-soft: #A8B8A0;
    --blue: #A8C3D6;
    --pink: #F4C2C2;
    --sand: #E8D5B7;
    --text: #4A4A4A;
    --text-light: #8B9A7C;
    --shadow: 0 8px 28px rgba(139, 154, 124, 0.12);
    --shadow-hover: 0 14px 35px rgba(139, 154, 124, 0.2);
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
  }

  /* ===== CONFETTI ===== */
  .confetti {
    position: fixed;
    top: -10px;
    pointer-events: none;
    z-index: 9999;
    border-radius: 2px;
    animation: confettiFall linear forwards;
  }

  @keyframes confettiFall {
    0% {
      transform: translateY(0) rotate(0deg);
      opacity: 1;
    }
    100% {
      transform: translateY(100vh) rotate(720deg);
      opacity: 0;
    }
  }

  /* ===== BURBUJAS ===== */
  .floating-bubble {
    position: fixed;
    width: var(--size);
    height: var(--size);
    top: var(--top);
    left: var(--left);
    right: var(--right);
    bottom: var(--bottom);
    pointer-events: none;
    z-index: 2;
    transform: translateY(var(--parallax));
    transition: transform 0.1s linear;
    animation: floatBubble var(--duration) ease-in-out infinite alternate;
    animation-delay: var(--delay);
    opacity: 0.5;
  }
  
  .bubble-content {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    font-size: calc(var(--size) * 0.5);
    filter: drop-shadow(0 4px 8px rgba(168, 195, 214, 0.2));
    transition: all 0.3s ease;
  }
  
  .floating-bubble:hover .bubble-content {
    transform: scale(1.1);
    opacity: 0.8;
  }

  @keyframes floatBubble {
    0% { transform: translateY(0) rotate(-5deg); }
    100% { transform: translateY(25px) rotate(5deg); }
  }

  /* ===== CONTENIDO PRINCIPAL ===== */
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
    text-shadow: 0 2px 6px rgba(139, 154, 124, 0.15);
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
  }
  
  .stat-badge {
    background: rgba(168, 195, 214, 0.2);
    padding: 5px 14px;
    border-radius: 40px;
    font-size: 0.8rem;
    font-weight: 600;
    color: var(--green);
    backdrop-filter: blur(4px);
  }

  /* ===== COMPOSITOR ===== */
  .composer {
    background: rgba(255, 255, 255, 0.85);
    backdrop-filter: blur(12px);
    border-radius: var(--radius-lg);
    padding: 22px;
    box-shadow: var(--shadow);
    margin-bottom: 32px;
    border: 2px solid rgba(255, 255, 255, 0.8);
    transition: all 0.3s ease;
  }
  
  .composer:hover {
    box-shadow: var(--shadow-hover);
    transform: translateY(-2px);
  }

  .preview-wrapper {
    position: relative;
    margin-bottom: 12px;
  }
  
  .preview-media {
    width: 100%;
    max-height: 220px;
    border-radius: var(--radius-md);
    object-fit: cover;
    box-shadow: 0 8px 24px rgba(0,0,0,0.1);
    border: 3px solid white;
  }
  
  .preview-audio {
    background: var(--sand);
    padding: 12px 16px;
    border-radius: var(--radius-md);
    font-size: 0.9rem;
    color: var(--text);
    text-align: center;
  }
  
  .remove-preview {
    position: absolute;
    top: 8px;
    right: 8px;
    background: rgba(255, 255, 255, 0.95);
    border: none;
    border-radius: 50%;
    width: 32px;
    height: 32px;
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
    color: var(--text);
    line-height: 1.6;
    transition: all 0.2s;
  }
  
  .composer-text:focus {
    outline: 3px solid var(--pink);
    outline-offset: 3px;
    background: white;
  }
  
  /* Emojis rápidos */
  .quick-emojis {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .emoji-btn {
    background: rgba(232, 213, 183, 0.4);
    border: none;
    font-size: 1.3rem;
    padding: 6px 12px;
    border-radius: 40px;
    cursor: pointer;
    transition: all 0.2s ease;
  }
  
  .emoji-btn:hover {
    transform: scale(1.15) translateY(-2px);
    background: rgba(232, 213, 183, 0.7);
  }

  /* Selector de colores */
  .color-selector {
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .color-label {
    font-size: 0.85rem;
    color: var(--text-light);
    font-weight: 600;
  }
  
  .color-options {
    display: flex;
    gap: 10px;
  }
  
  .color-option {
    width: 38px;
    height: 38px;
    border-radius: 50%;
    border: 3px solid white;
    box-shadow: 0 3px 10px rgba(0,0,0,0.1);
    cursor: pointer;
    transition: all 0.2s ease;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .color-option:hover {
    transform: scale(1.15);
    box-shadow: 0 5px 15px rgba(0,0,0,0.15);
  }
  
  .color-option.active {
    border-color: var(--green);
    transform: scale(1.1);
    box-shadow: 0 0 0 3px rgba(139, 154, 124, 0.3);
  }
  
  .checkmark {
    color: var(--green);
    font-weight: bold;
    font-size: 1rem;
    text-shadow: 0 1px 2px rgba(255,255,255,0.8);
  }

  .composer-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
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
    background: #8FB5CC; 
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(168, 195, 214, 0.4);
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
    box-shadow: 0 4px 12px rgba(139, 154, 124, 0.3);
  }
  
  .btn-send:hover:not(:disabled) { 
    transform: translateY(-2px) scale(1.02); 
    background: #7A8A6C;
    box-shadow: 0 8px 20px rgba(139, 154, 124, 0.4);
  }
  
  .btn-send:active:not(:disabled) {
    transform: scale(0.98);
  }
  
  .btn-send:disabled { 
    opacity: 0.6; 
    cursor: not-allowed; 
  }
  
  .btn-send.uploading {
    background: var(--sand);
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  @keyframes pulse {
    0%, 100% { opacity: 0.7; }
    50% { opacity: 1; }
  }

  /* ===== FEED ===== */
  .feed {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }
  
  .loader, .empty-state {
    text-align: center;
    padding: 40px 24px;
    background: rgba(255,255,255,0.7);
    backdrop-filter: blur(8px);
    border-radius: var(--radius-lg);
    border: 2px dashed var(--green-soft);
  }
  
  .loader {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
  }
  
  .loader-emoji {
    display: inline-block;
    animation: bounce 1s ease-in-out infinite;
  }
  
  .empty-emoji {
    font-size: 3rem;
    display: block;
    margin-bottom: 12px;
  }
  
  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-5px); }
  }

  /* ===== MASCOTA ===== */
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
    filter: drop-shadow(0 6px 12px rgba(0,0,0,0.1));
    transition: transform 0.2s;
  }
  
  .mascot:hover {
    transform: scale(1.05);
  }
  
  .mascot:hover + .mascot-tooltip {
    opacity: 1;
    transform: translateY(0);
  }
  
  .mascot-tooltip {
    position: absolute;
    bottom: 75px;
    right: 0;
    background: white;
    padding: 6px 14px;
    border-radius: 30px;
    font-size: 0.8rem;
    font-weight: 600;
    color: var(--green);
    white-space: nowrap;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
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

  /* ===== RESPONSIVE ===== */
  @media (max-width: 560px) {
    .logo { font-size: 2.5rem; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .color-selector { justify-content: center; }
    .quick-emojis { justify-content: center; }
    .mascot { width: 55px; }
    .stats { flex-direction: column; align-items: center; gap: 8px; }
    .composer { padding: 16px; }
  }
</style>