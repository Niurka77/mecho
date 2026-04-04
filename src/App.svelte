<script>
  import { onMount, onDestroy } from 'svelte';
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

  // --- CARGAR POSTS + TIEMPO REAL ---
  onMount(async () => {
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) console.error('Error cargando posts:', error);
    else posts = data || [];
    loading = false;

    // Suscribirse a nuevos posts en tiempo real
    channel = supabase
      .channel('mecho-realtime')
      .on('postgres_changes', 
        { event: 'INSERT', schema: 'public', table: 'posts' }, 
        payload => {
          posts = [payload.new, ...posts];
        }
      )
      .subscribe();

    // Escuchar scroll para parallax de burbujas
    window.addEventListener('scroll', handleScroll);
  });

  onDestroy(() => {
    if (channel) supabase.removeChannel(channel);
    window.removeEventListener('scroll', handleScroll);
  });

  function handleScroll() {
    scrollY = window.scrollY;
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

  // --- ENVIAR POST MIXTO ---
  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✨ Escribe algo o adjunta un recuerdo');
      return;
    }

    let currentMediaUrl = null;

    // Subir archivo si existe
    if (mediaFile) {
      const fileExt = mediaFile.name.split('.').pop();
      const safeName = `${Date.now()}_${Math.random().toString(36).slice(2, 8)}.${fileExt}`;
      const datePath = new Date().toISOString().split('T')[0];
      const filePath = `posts/${datePath}/${safeName}`;

      const { data, error: uploadError } = await supabase.storage
        .from('mecho-media')
        .upload(filePath, mediaFile, { cacheControl: '3600' });

      if (uploadError) {
        console.error('Error subiendo:', uploadError);
        alert('No se pudo subir el archivo. Intenta de nuevo.');
        return;
      }
      
      const { data: { publicUrl } } = supabase.storage
        .from('mecho-media')
        .getPublicUrl(filePath);
      currentMediaUrl = publicUrl;
    }

    // Crear post mixto (texto + media en uno)
    const newPostData = {
      text: textInput.trim() || null,
      media_url: currentMediaUrl || null,
      type: mediaFile 
        ? (mediaFile.type.startsWith('image') ? 'image' 
          : mediaFile.type.startsWith('video') ? 'video' 
          : mediaFile.type.startsWith('audio') ? 'audio' 
          : 'note') 
        : 'note'
    };

    const { error: dbError } = await supabase
      .from('posts')
      .insert([newPostData]);

    if (dbError) {
      console.error('Error en BD:', dbError);
      alert('Algo salió mal. Revisa tu conexión.');
    } else {
      // Limpiar formulario
      textInput = '';
      mediaFile = null;
      mediaPreview = null;
      if (fileInputRef) fileInputRef.value = '';
    }
  }

  // --- POSICIONES ALEATORIAS PARA BURBUJAS (fijas, no tapan contenido) ---
  const bubbles = [
    { top: '15%', left: '8%', size: 50, delay: 0, duration: 18 },
    { top: '70%', right: '6%', size: 40, delay: 2, duration: 22 },
    { bottom: '20%', left: '4%', size: 35, delay: 4, duration: 20 },
  ];
</script>

<div class="app-wrapper">
  
  <!-- Burbujas flotantes sutiles (no tapan contenido) -->
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

  <!-- Burbuja principal con Mew (parallax al scroll) -->
  <div class="mew-bubble" style="--parallax: {scrollY * 0.08}px;">
    <div class="bubble-core">
      <span class="mew-pair">🤍🫧</span>
    </div>
  </div>

  <main class="content">
    <header>
      <h1 class="logo">mecho 🌿</h1>
      <p class="subtitle">un rincón suave para nosotras</p>
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
            <div class="preview-audio">🎵 Audio listo: {mediaFile.name.slice(0, 15)}...</div>
          {/if}
          <button class="remove-preview" on:click={() => { 
            mediaPreview = null; mediaFile = null; 
            if (fileInputRef) fileInputRef.value = ''; 
          }}>&times;</button>
        </div>
      {/if}

      <textarea 
        bind:value={textInput} 
        placeholder="Escribe un mensajito bonito... ✏️"
        rows="3"
        class="composer-text"
      ></textarea>
      
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
          📎 {mediaFile ? mediaFile.name.slice(0, 18) + (mediaFile.name.length > 18 ? '...' : '') : 'Adjuntar'}
        </label>
        <button 
          class="btn-send" 
          on:click={sendPost} 
          disabled={!textInput.trim() && !mediaFile}
        >
          Enviar ✨
        </button>
      </div>
    </section>

    <!-- Feed de posts -->
    <section class="feed">
      {#if loading}
        <div class="loader">Mecho está despertando... 🌸</div>
      {:else if posts.length === 0}
        <div class="empty-state">Aún no hay recuerdos. ¡Sé la primera en compartir! 💫</div>
      {:else}
        {#each posts as post (post.id)}
          <Post {post} />
        {/each}
      {/if}
    </section>
  </main>

  <!-- Mascota flotante en esquina -->
  <img src="/mecho.png" alt="Mecha" class="mascot" />
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&family=VT323&display=swap');

  :root {
    --bg: #F5F9F6;
    --card: #FFFFFF;
    --green: #8B9A7C;
    --green-soft: #A8B8A0;
    --blue: #A8C3D6;
    --pink: #F4C2C2;
    --sand: #E8D5B7;
    --text: #4A4A4A;
    --text-light: #7A7A7A;
    --shadow: 0 6px 20px rgba(139, 154, 124, 0.12);
    --shadow-hover: 0 10px 28px rgba(139, 154, 124, 0.18);
    --radius-lg: 30px;
    --radius-md: 22px;
    --radius-sm: 16px;
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }
  
  body {
    background: var(--bg);
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

  /* ===== BURBUJAS FLOTANTES (sutiles, no tapan contenido) ===== */
  .floating-bubble {
    position: fixed;
    width: var(--size);
    height: var(--size);
    top: var(--top);
    left: var(--left);
    right: var(--right);
    bottom: var(--bottom);
    pointer-events: none;
    z-index: 3;
    transform: translateY(var(--parallax));
    transition: transform 0.1s linear;
    animation: floatBubble var(--duration) ease-in-out infinite alternate;
    animation-delay: var(--delay);
    opacity: 0.7;
  }
  
  .bubble-content {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    font-size: calc(var(--size) * 0.5);
    filter: drop-shadow(0 4px 8px rgba(168, 195, 214, 0.2));
  }

  @keyframes floatBubble {
    0% { transform: translateY(0) rotate(-3deg); }
    100% { transform: translateY(20px) rotate(3deg); }
  }

  /* Burbuja principal con Mew */
  .mew-bubble {
    position: fixed;
    top: 12%;
    left: 12px;
    width: 90px;
    height: 110px;
    z-index: 5;
    pointer-events: none;
    transform: translateY(var(--parallax));
    transition: transform 0.1s linear;
  }
  .bubble-core {
    background: rgba(255, 255, 255, 0.7);
    border-radius: 50%;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 2rem;
    box-shadow: 0 10px 30px rgba(168, 195, 214, 0.25);
    animation: bubbleDrift 14s ease-in-out infinite alternate;
    border: 2px solid rgba(255, 255, 255, 0.4);
  }
  @keyframes bubbleDrift {
    0% { transform: translateY(0) rotate(-6deg); }
    100% { transform: translateY(35px) rotate(6deg); }
  }

  /* ===== CONTENIDO PRINCIPAL ===== */
  .content {
    max-width: 640px;
    margin: 0 auto;
    padding: 24px 16px 100px;
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
    font-size: 2.8rem;
    color: var(--green);
    letter-spacing: 3px;
    text-shadow: 0 2px 4px rgba(0,0,0,0.05);
  }
  .subtitle {
    color: var(--text-light);
    font-size: 1.05rem;
    margin-top: 6px;
    font-weight: 500;
  }

  /* ===== COMPOSITOR DE POSTS ===== */
  .composer {
    background: var(--card);
    border-radius: var(--radius-lg);
    padding: 20px;
    box-shadow: var(--shadow);
    margin-bottom: 32px;
    border: 2px solid var(--sand);
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  .preview-wrapper {
    position: relative;
    margin-bottom: 8px;
  }
  .preview-media {
    width: 100%;
    max-height: 220px;
    border-radius: var(--radius-md);
    object-fit: cover;
    box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  }
  .preview-audio {
    background: var(--sand);
    padding: 12px 16px;
    border-radius: var(--radius-md);
    font-size: 0.95rem;
    color: var(--text);
    text-align: center;
  }
  .remove-preview {
    position: absolute;
    top: 8px;
    right: 8px;
    background: rgba(255, 255, 255, 0.9);
    border: none;
    border-radius: 50%;
    width: 28px;
    height: 28px;
    font-size: 1.3rem;
    color: var(--text);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 3px 8px rgba(0,0,0,0.1);
    transition: background 0.2s;
  }
  .remove-preview:hover { background: white; }

  .composer-text {
    width: 100%;
    border: none;
    background: var(--sand);
    border-radius: var(--radius-md);
    padding: 14px 16px;
    font-family: 'Nunito', sans-serif;
    font-size: 1.05rem;
    resize: vertical;
    color: var(--text);
    line-height: 1.5;
  }
  .composer-text:focus {
    outline: 2px solid var(--pink);
    outline-offset: 2px;
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
    padding: 10px 16px;
    border-radius: 24px;
    font-weight: 600;
    font-size: 0.95rem;
    cursor: pointer;
    transition: background 0.2s, transform 0.1s;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 200px;
  }
  .btn-attach:hover { background: #8FB5CC; transform: translateY(-1px); }

  .btn-send {
    background: var(--green);
    color: white;
    border: none;
    padding: 12px 28px;
    border-radius: 28px;
    font-family: 'Nunito', sans-serif;
    font-weight: 700;
    font-size: 1.05rem;
    cursor: pointer;
    transition: transform 0.15s, opacity 0.2s, background 0.2s;
    box-shadow: 0 4px 12px rgba(139, 154, 124, 0.2);
  }
  .btn-send:hover:not(:disabled) { 
    transform: translateY(-2px); 
    background: #7A8A6C;
    box-shadow: 0 6px 16px rgba(139, 154, 124, 0.28);
  }
  .btn-send:disabled { 
    opacity: 0.6; 
    cursor: not-allowed; 
    transform: none;
  }

  /* ===== FEED DE POSTS ===== */
  .feed {
    display: flex;
    flex-direction: column;
    gap: 18px;
  }
  .loader, .empty-state {
    text-align: center;
    color: var(--text-light);
    padding: 24px;
    font-size: 1.05rem;
    background: rgba(255,255,255,0.6);
    border-radius: var(--radius-md);
    border: 1px dashed var(--green-soft);
  }

  /* ===== MASCOTA FLOTANTE ===== */
  .mascot {
    position: fixed;
    bottom: 24px;
    right: 18px;
    width: 64px;
    height: auto;
    animation: floatMascot 4s ease-in-out infinite;
    z-index: 20;
    filter: drop-shadow(0 4px 8px rgba(0,0,0,0.1));
  }
  @keyframes floatMascot {
    0%, 100% { transform: translateY(0) rotate(-2deg); }
    50% { transform: translateY(-12px) rotate(2deg); }
  }

  /* ===== RESPONSIVE ===== */
  @media (max-width: 480px) {
    .logo { font-size: 2.4rem; }
    .composer-actions { flex-direction: column; align-items: stretch; }
    .btn-attach, .btn-send { width: 100%; text-align: center; }
    .mew-bubble { width: 70px; height: 85px; top: 10%; left: 8px; }
    .bubble-core { font-size: 1.6rem; }
  }
</style>