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
  
  // Burbujas
  let bubbles = [];
  let popSound;
  
  // Sonido
  let sendSound;
  
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

  // Configuración de burbujas realistas
  const bubbleConfigs = [
    { size: 80, left: '10%', duration: 8, delay: 0, shadow: '#ff9a9e' },
    { size: 40, left: '20%', duration: 5, delay: 1, shadow: '#fad0c4' },
    { size: 100, left: '35%', duration: 12, delay: 2, shadow: '#a18cd1' },
    { size: 60, left: '50%', duration: 7, delay: 0.5, shadow: '#fbc2eb' },
    { size: 120, left: '65%', duration: 15, delay: 3, shadow: '#8fd3f4' },
    { size: 50, left: '80%', duration: 9, delay: 1.5, shadow: '#cfd9df' },
    { size: 90, left: '15%', duration: 11, delay: 4, shadow: '#a6c1ee' },
    { size: 30, left: '45%', duration: 6, delay: 2.5, shadow: '#ffecd2' },
    { size: 70, left: '75%', duration: 10, delay: 5, shadow: '#d4fc79' },
    { size: 110, left: '90%', duration: 14, delay: 1, shadow: '#96e6a1' },
    { size: 55, left: '25%', duration: 13, delay: 3.5, shadow: '#ffdde1' },
    { size: 45, left: '55%', duration: 6.5, delay: 4.2, shadow: '#ee9ca7' },
    { size: 85, left: '70%', duration: 9.5, delay: 2.8, shadow: '#ffd89b' },
    { size: 35, left: '8%', duration: 7.5, delay: 5.5, shadow: '#c9ffbf' }
  ];

  // --- CARGAR POSTS + TIEMPO REAL ---
  onMount(async () => {
    // Cargar sonidos
    sendSound = new Audio('/soft-pop.mp3');
    sendSound.volume = 0.15;
    
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.2;
    
    // Inicializar burbujas con IDs únicos
    bubbles = bubbleConfigs.map((config, index) => ({
      id: index,
      ...config,
      key: Date.now() + index,
      exploding: false
    }));
    
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
          if (sendSound) sendSound.play();
        }
      )
      .subscribe();

    window.addEventListener('scroll', handleScroll);
    
    // Regenerar burbujas periódicamente
    const interval = setInterval(() => {
      regenerateBubbles();
    }, 15000);
    
    return () => clearInterval(interval);
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

  // Regenerar burbujas que se fueron
  function regenerateBubbles() {
    bubbles = bubbles.map(bubble => ({
      ...bubble,
      key: Date.now() + bubble.id,
      exploding: false
    }));
  }

  // --- EXPLOTAR BURBUJA ---
  function popBubble(bubbleId) {
    const bubble = bubbles.find(b => b.id === bubbleId);
    if (!bubble || bubble.exploding) return;
    
    // Reproducir sonido
    if (popSound) {
      popSound.currentTime = 0;
      popSound.play();
    }
    
    // Marcar como explotando
    bubbles = bubbles.map(b => 
      b.id === bubbleId ? { ...b, exploding: true, key: Date.now() } : b
    );
    
    // Regenerar la burbuja después de 2 segundos
    setTimeout(() => {
      bubbles = bubbles.map(b =>
        b.id === bubbleId ? { ...b, exploding: false, key: Date.now() } : b
      );
    }, 2000);
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
      createConfetti();
      if (sendSound) sendSound.play();
      
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

  // Manejar like
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
</script>

<div class="app-wrapper">
  
  <!-- Burbujas realistas interactivas -->
  {#each bubbles as bubble}
    <div
      key={bubble.key}
      class="bubble {bubble.exploding ? 'exploding' : ''}"
      style="
        width: {bubble.size}px;
        height: {bubble.size}px;
        left: {bubble.left};
        animation: floatUp {bubble.duration}s infinite ease-in;
        animation-delay: {bubble.delay}s;
        box-shadow: inset 0 0 {bubble.size * 0.3}px {bubble.shadow};
      "
      on:click={() => popBubble(bubble.id)}
      on:touchstart|preventDefault={() => popBubble(bubble.id)}
    >
      <div class="bubble-shine"></div>
      {#if bubble.exploding}
        <div class="pop-particles">
          {#each Array(8) as _, i}
            <span class="particle" style="--angle: {i * 45}deg; --delay: {i * 0.05}s">✨</span>
          {/each}
        </div>
      {/if}
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
    <div class="mascot-tooltip">¡toca las burbujas! 💖</div>
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;500;600;700;800&family=VT323&display=swap');

  :root {
    --bg: #FDFBF5;
    --bg-gradient: linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%);
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
    overflow-x: hidden;
  }

  /* ===== BURBUJAS REALISTAS ===== */
  .bubble {
    position: fixed;
    bottom: -150px;
    background: rgba(255, 255, 255, 0.15);
    border-radius: 50%;
    backdrop-filter: blur(1px);
    pointer-events: auto;
    cursor: pointer;
    transition: all 0.1s ease;
    z-index: 5;
  }
  
  .bubble:hover {
    transform: scale(1.05);
  }
  
  .bubble-shine {
    position: absolute;
    top: 15%;
    left: 15%;
    width: 25%;
    height: 20%;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.7);
    transform: rotate(-30deg);
    filter: blur(1px);
  }
  
  /* Animación de ascenso */
  @keyframes floatUp {
    0% {
      transform: translateY(0) translateX(0);
      opacity: 0;
    }
    10% {
      opacity: 0.7;
    }
    50% {
      transform: translateY(-50vh) translateX(30px);
    }
    100% {
      transform: translateY(-120vh) translateX(-20px);
      opacity: 0;
    }
  }
  
  /* Animación de explosión */
  .bubble.exploding {
    animation: pop 0.3s ease-out forwards !important;
    pointer-events: none;
  }
  
  @keyframes pop {
    0% {
      transform: scale(1);
      opacity: 0.8;
    }
    50% {
      transform: scale(1.3);
      opacity: 0.5;
    }
    100% {
      transform: scale(0);
      opacity: 0;
    }
  }
  
  /* Partículas al explotar */
  .pop-particles {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 100%;
    height: 100%;
    transform: translate(-50%, -50%);
    pointer-events: none;
  }
  
  .particle {
    position: absolute;
    top: 50%;
    left: 50%;
    font-size: 14px;
    animation: particleFly 0.5s ease-out forwards;
    transform: rotate(var(--angle)) translateX(20px);
  }
  
  @keyframes particleFly {
    0% {
      opacity: 1;
      transform: rotate(var(--angle)) translateX(0) scale(1);
    }
    100% {
      opacity: 0;
      transform: rotate(var(--angle)) translateX(40px) scale(0);
    }
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
    color: white;
    letter-spacing: 4px;
    text-shadow: 0 2px 10px rgba(0,0,0,0.15);
  }
  
  .subtitle {
    color: rgba(255,255,255,0.9);
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
    background: rgba(255,255,255,0.25);
    backdrop-filter: blur(8px);
    padding: 5px 14px;
    border-radius: 40px;
    font-size: 0.8rem;
    font-weight: 600;
    color: white;
  }

  /* ===== COMPOSITOR ===== */
  .composer {
    background: rgba(255, 255, 255, 0.9);
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
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .color-option:hover {
    transform: scale(1.15);
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
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  
  .btn-attach:hover { 
    background: #8FB5CC; 
    transform: translateY(-2px);
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
  }
  
  .btn-send:hover:not(:disabled) { 
    transform: translateY(-2px) scale(1.02); 
    background: #7A8A6C;
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
  }
  
  @keyframes floatMascot {
    0%, 100% { transform: translateY(0) rotate(-3deg); }
    50% { transform: translateY(-12px) rotate(3deg); }
  }

  @media (max-width: 560px) {
    .logo { font-size: 2.5rem; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .color-selector { justify-content: center; }
    .quick-emojis { justify-content: center; }
    .mascot { width: 55px; }
    .stats { flex-direction: column; align-items: center; gap: 8px; }
  }
</style>