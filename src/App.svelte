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
  
  // Nuevo estado para colores y upload
  let selectedColor = '#FFFFFF';
  let isUploading = false;
  
  // Colores pastel disponibles
  const pastelColors = [
    { value: '#FFE5E5', label: 'Rosa' },    // Rosa pastel
    { value: '#E5F3FF', label: 'Azul' },    // Azul pastel
    { value: '#E5FFE5', label: 'Verde' },   // Verde pastel
    { value: '#FFF9E5', label: 'Amarillo' } // Amarillo pastel
  ];

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

    isUploading = true;
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
        isUploading = false;
        return;
      }
      
      const { data: { publicUrl } } = supabase.storage
        .from('mecho-media')
        .getPublicUrl(filePath);
      currentMediaUrl = publicUrl;
    }

    // Crear post mixto (texto + media + color)
    const newPostData = {
      text: textInput.trim() || null,
      media_url: currentMediaUrl || null,
      color: selectedColor,
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
      selectedColor = '#FFFFFF';
      if (fileInputRef) fileInputRef.value = '';
    }
    
    isUploading = false;
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

 

  <main class="content">
    <header>
      <h1 class="logo">mecho </h1>
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
      
      <!-- Selector de colores pastel -->
      <div class="color-selector">
        <span class="color-label">Elige un color:</span>
        <div class="color-options">
          {#each pastelColors as colorOption}
            <button
              class="color-option {selectedColor === colorOption.value ? 'active' : ''}"
              style="background-color: {colorOption.value}"
              on:click={() => selectedColor = colorOption.value}
              title={colorOption.label}
              aria-label={`Seleccionar color ${colorOption.label}`}
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
          📎 {mediaFile ? mediaFile.name.slice(0, 18) + (mediaFile.name.length > 18 ? '...' : '') : 'Adjuntar'}
        </label>
        <button 
          class="btn-send {isUploading ? 'uploading' : ''}" 
          on:click={sendPost} 
          disabled={!textInput.trim() && !mediaFile || isUploading}
        >
          {#if isUploading}
            Mecho está trabajando... 🐾
          {:else}
            Enviar ✨
          {/if}
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
          <div transition:fly={{ y: 20, duration: 400, opacity: 0 }}>
            <Post {post} />
          </div>
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
    --bg: #FDFBF5;
    --bg-gradient: linear-gradient(180deg, #FDFBF5 0%, #F9F7F0 100%);
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
    box-shadow: 0 8px 24px rgba(0,0,0,0.12);
    border: 3px solid white;
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
    background: rgba(255, 255, 255, 0.95);
    border: none;
    border-radius: 50%;
    width: 32px;
    height: 32px;
    font-size: 1.4rem;
    color: var(--text);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    transition: all 0.2s;
  }
  .remove-preview:hover { 
    background: white; 
    transform: scale(1.1);
    box-shadow: 0 6px 16px rgba(0,0,0,0.2);
  }

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
    outline: 3px solid var(--pink);
    outline-offset: 2px;
  }

  /* ===== SELECTOR DE COLORES ===== */
  .color-selector {
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
  }
  
  .color-label {
    font-size: 0.9rem;
    color: var(--text-light);
    font-weight: 600;
  }
  
  .color-options {
    display: flex;
    gap: 10px;
  }
  
  .color-option {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    border: 3px solid white;
    box-shadow: 0 3px 10px rgba(0,0,0,0.15);
    cursor: pointer;
    transition: all 0.2s ease;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .color-option:hover {
    transform: scale(1.15);
    box-shadow: 0 5px 15px rgba(0,0,0,0.2);
  }
  
  .color-option.active {
    border-color: var(--green);
    transform: scale(1.1);
    box-shadow: 0 0 0 3px rgba(139, 154, 124, 0.3);
  }
  
  .checkmark {
    color: var(--green);
    font-weight: bold;
    font-size: 1.1rem;
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
    padding: 10px 16px;
    border-radius: 24px;
    font-weight: 600;
    font-size: 0.95rem;
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
    box-shadow: 0 4px 12px rgba(168, 195, 214, 0.3);
  }

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
    transition: all 0.2s;
    box-shadow: 0 4px 12px rgba(139, 154, 124, 0.25);
  }
  .btn-send:hover:not(:disabled) { 
    transform: translateY(-2px); 
    background: #7A8A6C;
    box-shadow: 0 8px 20px rgba(139, 154, 124, 0.35);
  }
  .btn-send:disabled { 
    opacity: 0.65; 
    cursor: not-allowed; 
    transform: none;
  }
  .btn-send.uploading {
    background: var(--sand);
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  @keyframes pulse {
    0%, 100% { opacity: 0.65; }
    50% { opacity: 1; }
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
    background: rgba(255,255,255,0.7);
    border-radius: var(--radius-md);
    border: 2px dashed var(--green-soft);
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
    .color-selector { justify-content: center; }
  }
</style>