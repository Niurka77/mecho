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
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) console.error('Error cargando posts:', error);
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

  function handleScroll() {
    scrollY = window.scrollY;
  }
  
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
    } else {
      mediaPreview = null;
    }
  }

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

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✨ Escribe algo o adjunta un recuerdo');
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
      if (popSound) popSound.play();
      
      textInput = '';
      mediaFile = null;
      mediaPreview = null;
      selectedColor = '#FFFFFF';
      if (fileInputRef) fileInputRef.value = '';
    }
    
    isUploading = false;
  }

  function addEmoji(emoji) {
    textInput += emoji;
  }

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

  // Burbujas con posición y animación personalizada
  const bubbles = [
    { left: '10%', delay: '0s', duration: '8s', size: '80px', color: '#ff9a9e' },
    { left: '20%', delay: '1s', duration: '5s', size: '40px', color: '#fad0c4' },
    { left: '35%', delay: '2s', duration: '12s', size: '100px', color: '#a18cd1' },
    { left: '50%', delay: '0.5s', duration: '7s', size: '60px', color: '#fbc2eb' },
    { left: '65%', delay: '3s', duration: '15s', size: '120px', color: '#8fd3f4' },
    { left: '80%', delay: '1.5s', duration: '9s', size: '50px', color: '#cfd9df' },
    { left: '15%', delay: '4s', duration: '11s', size: '90px', color: '#a6c1ee' },
    { left: '45%', delay: '2.5s', duration: '6s', size: '30px', color: '#ffecd2' },
    { left: '75%', delay: '5s', duration: '10s', size: '70px', color: '#d4fc79' },
    { left: '90%', delay: '1s', duration: '14s', size: '110px', color: '#96e6a1' }
  ];
</script>

<div class="app-wrapper">
  
  <!-- Burbujas realistas CSS -->
  {#each bubbles as bubble, i}
    <div 
      class="bubble" 
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

  <!-- Confetti -->
  {#if showConfetti}
    <div class="confetti-container"></div>
  {/if}

  <main class="content">
    <header>
      <h1 class="logo">mecho </h1>
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
          <span class="empty-bubble"></span>
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
    overflow: hidden;
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
    0% { transform: translateY(0) rotate(0deg); opacity: 1; }
    100% { transform: translateY(100vh) rotate(720deg); opacity: 0; }
  }

  /* Busca la clase .bubble en tu <style> y reemplázala con esto */
.bubble {
    position: fixed;
    bottom: -150px;
    left: var(--bubble-left);
    width: var(--bubble-size);
    height: var(--bubble-size);
    
    /* MEJORA 1: Fondo más visible (0.3 en lugar de 0.1) */
    background: rgba(255, 255, 255, 0.3); 
    
    /* MEJORA 2: Un borde blanco muy fino para definir la forma */
    border: 1px solid rgba(255, 255, 255, 0.5); 
    
    border-radius: 50%;
    box-shadow: 
      0 20px 30px rgba(0, 0, 0, 0.05),
      inset 0 10px 30px 5px rgba(255, 255, 255, 1), /* Brillo interno más fuerte */
      inset 0 -15px 30px 0px rgba(255, 255, 255, 0.6),
      inset 0 0 25px var(--bubble-color); /* Color un poco más intenso */
    
    backdrop-filter: blur(2px); /* Un poquito más de desenfoque interno */
    pointer-events: none;
    z-index: 1;
    transform: translateY(var(--parallax-y));
    animation: floatUp var(--bubble-duration) infinite ease-in;
    animation-delay: var(--bubble-delay);
    will-change: transform;
}
/* Busca .bubble::after y ajusta el background */
.bubble::after {
    content: "";
    position: absolute;
    top: 15%;
    left: 15%;
    width: 25%;
    height: 20%;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.9); /* Subimos de 0.6 a 0.9 */
    transform: rotate(-30deg);
    filter: blur(1px);
}

@keyframes floatUp {
    0% { transform: translateY(0) translateX(0); opacity: 0; }
    15% { opacity: 0.8; } /* Subimos de 0.6 a 0.8 para que se noten rápido */
    80% { opacity: 0.8; }
    100% { transform: translateY(-120vh) translateX(-15px); opacity: 0; }
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

  .preview-wrapper { position: relative; margin-bottom: 12px; }
  
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
  
  .btn-send:active:not(:disabled) { transform: scale(0.98); }
  .btn-send:disabled { opacity: 0.6; cursor: not-allowed; }
  .btn-send.uploading {
    background: var(--sand);
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  @keyframes pulse { 0%, 100% { opacity: 0.7; } 50% { opacity: 1; } }

  /* ===== FEED ===== */
  .feed { display: flex; flex-direction: column; gap: 20px; }
  
  .loader, .empty-state {
    text-align: center;
    padding: 40px 24px;
    background: rgba(255,255,255,0.7);
    backdrop-filter: blur(8px);
    border-radius: var(--radius-lg);
    border: 2px dashed var(--green-soft);
  }
  
  .loader { display: flex; align-items: center; justify-content: center; gap: 10px; }
  .loader-emoji { display: inline-block; animation: bounce 1s ease-in-out infinite; }
  
  .empty-bubble {
    width: 50px;
    height: 50px;
    margin: 0 auto 12px;
    background: rgba(255,255,255,0.3);
    border-radius: 50%;
    box-shadow: inset 0 10px 20px rgba(255,255,255,0.6), inset 0 0 15px #a18cd1;
    position: relative;
  }
  .empty-bubble::after {
    content: "";
    position: absolute;
    top: 15%; left: 15%;
    width: 25%; height: 20%;
    border-radius: 50%;
    background: rgba(255,255,255,0.6);
    transform: rotate(-30deg);
    filter: blur(1px);
  }
  
  @keyframes bounce { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-5px); } }

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
  
  .mascot:hover { transform: scale(1.05); }
  .mascot:hover + .mascot-tooltip { opacity: 1; transform: translateY(0); }
  
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
  @media (max-width: 768px) {
    .bubble { --bubble-size: calc(var(--bubble-size) * 0.7) !important; }
    @keyframes floatUp {
      0% { transform: translateY(0) translateX(0); opacity: 0; }
      10% { opacity: 0.5; }
      50% { transform: translateY(-40vh) translateX(20px); }
      100% { transform: translateY(-110vh) translateX(-10px); opacity: 0; }
    }
  }

  @media (max-width: 560px) {
    .logo { font-size: 2.5rem; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .color-selector { justify-content: center; }
    .quick-emojis { justify-content: center; }
    .mascot { width: 55px; }
    .stats { flex-direction: column; align-items: center; gap: 8px; }
    .composer { padding: 16px; }
    .bubble { --bubble-size: calc(var(--bubble-size) * 0.5) !important; opacity: 0.4; }
  }

  @media (max-width: 380px) {
    .bubble { display: none; } /* Ocultar burbujas en pantallas muy pequeñas para rendimiento */
  }
</style>