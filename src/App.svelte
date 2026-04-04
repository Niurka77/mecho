<script>
  import { onMount, onDestroy } from 'svelte';
  import { fly } from 'svelte/transition';
  import { supabase } from './lib/supabaseClient';
  import Post from './lib/Post.svelte';

  let posts = [];
  let loading = true;
  
  // Estado del formulario
  let textInput = '';
  let authorName = ''; // Nuevo campo
  let mediaFile = null;
  let mediaPreview = null;
  let fileInputRef;
  let textareaRef;
  
  let scrollY = 0;
  let channel;
  let isUploading = false;
  let showConfetti = false;
  let todayCount = 0;
  let popSound;
  
  // Moods simplificados (solo colores sutiles para el guestbook)
  const moods = [
    { label: 'Normal', color: '#FFFFFF' },
    { label: 'Suave', color: '#FFF0F5' }, // Lavender Blush
    { label: 'Cielo', color: '#F0F8FF' }, // Alice Blue
    { label: 'Menta', color: '#F0FFF0' }  // Honeydew
  ];

  let selectedMood = moods[0];

  onMount(async () => {
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    // Cargar posts
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .order('created_at', { ascending: false });
    
    if (error) console.error('Error:', error);
    else posts = data || [];
    loading = false;
    updateTodayCount();

    // Realtime
    channel = supabase
      .channel('mecho-realtime')
      .on('postgres_changes', 
        { event: 'INSERT', schema: 'public', table: 'posts' }, 
        payload => {
          posts = [payload.new, ...posts];
          updateTodayCount();
          if (popSound) popSound.play().catch(() => {});
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

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) return;
    if (!authorName.trim()) {
      alert('Por favor, firma tu mensaje.');
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

      if (!uploadError) {
        const { data: { publicUrl } } = supabase.storage.from('mecho-media').getPublicUrl(filePath);
        currentMediaUrl = publicUrl;
      }
    }

    let postType = 'note';
    if (mediaFile) {
      if (mediaFile.type.startsWith('image')) postType = 'image';
      else if (mediaFile.type.startsWith('video')) postType = 'video';
      else if (mediaFile.type.startsWith('audio')) postType = 'audio';
    }

    const newPostData = {
      text: textInput.trim(),
      author_name: authorName.trim(), // Guardar nombre
      media_url: currentMediaUrl,
      mood_color: selectedMood.color, // Guardamos solo el color de fondo sutil
      type: postType,
      likes: 0
    };

    const { error: dbError } = await supabase.from('posts').insert([newPostData]);
    
    if (dbError) {
      console.error('Error BD:', dbError);
      alert('Error al guardar');
    } else {
      textInput = ''; 
      authorName = '';
      mediaFile = null; 
      mediaPreview = null;
      selectedMood = moods[0];
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

  // Burbujas simplificadas y enviadas al fondo
  const bubbles = Array.from({ length: 8 }).map((_, i) => ({
    left: Math.random() * 90 + '%',
    delay: Math.random() * 5 + 's',
    duration: 10 + Math.random() * 10 + 's',
    size: 40 + Math.random() * 60 + 'px',
    color: ['#ffb3c1', '#b3e0f2', '#d4c1ec'][Math.floor(Math.random() * 3)]
  }));
</script>

<div class="app-wrapper">
  <!-- Fondo fijo -->
  <div class="bg-image"></div>

  <!-- Burbujas detrás del contenido (z-index bajo) -->
  <div class="bubble-layer">
    {#each bubbles as bubble, i}
      <div class="bubble" 
        style="
          --size: {bubble.size};
          --left: {bubble.left};
          --delay: {bubble.delay};
          --dur: {bubble.duration};
          --color: {bubble.color};
        "
      ></div>
    {/each}
  </div>

  <main class="content">
    <header class="retro-header">
      <h1 class="logo">GUESTBOOK</h1>
      <p class="subtitle">deja tu huella suave</p>
    </header>

    <!-- PANEL PRINCIPAL (Cuadro con borde doble) -->
    <section class="main-panel">
      
      <!-- Área de Input Retro -->
      <div class="input-window">
        <div class="window-title-bar">
          <span>nuevo_mensaje.txt</span>
          <div class="window-controls"><div></div><div></div></div>
        </div>
        
        <div class="window-content">
          <input 
            type="text" 
            bind:value={authorName}
            placeholder="tu nombre..." 
            class="retro-input-name"
            maxlength="20"
          />
          
          <textarea 
            bind:value={textInput} 
            bind:this={textareaRef}
            placeholder="escribe algo aquí..."
            rows="4"
            class="retro-textarea"
          ></textarea>

          <div class="input-footer">
            <div class="mood-selector-mini">
              {#each moods as mood}
                <button 
                  class="mood-dot {selectedMood === mood ? 'active' : ''}"
                  style="background-color: {mood.color}; border: 1px solid #ccc"
                  on:click={() => selectedMood = mood}
                  title={mood.label}
                ></button>
              {/each}
            </div>

            <div class="actions">
              <input 
                type="file" 
                accept="image/*,video/*" 
                bind:this={fileInputRef} 
                on:change={handleFileChange} 
                class="file-input-hidden"
                id="media-input"
              />
              <label for="media-input" class="btn-retro-attach">
                📎
              </label>
              <button 
                class="btn-retro-send" 
                on:click={sendPost} 
                disabled={(!textInput.trim() && !mediaFile) || isUploading}
              >
                {#if isUploading}...{:else}ENVIAR{/if}
              </button>
            </div>
          </div>
          
          {#if mediaPreview}
            <div class="mini-preview">
              <img src={mediaPreview} alt="preview" />
              <button on:click={() => { mediaPreview=null; mediaFile=null; }}>x</button>
            </div>
          {/if}
        </div>
      </div>

      <!-- FEED DE MENSAJES -->
      <section class="feed-container">
        {#if loading}
          <div class="loader-text">cargando memorias...</div>
        {:else if posts.length === 0}
          <div class="empty-msg">el libro está vacío.</div>
        {:else}
          {#each posts as post (post.id)}
            <div transition:fly={{ y: 10, duration: 400 }}>
              <Post {post} onLike={handleLike} />
            </div>
          {/each}
        {/if}
      </section>
    </section>
  </main>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=VT323&family=Nunito:wght@400;600&display=swap');

  :root {
    --pink: #D4A5A5;
    --blue: #9DB5C7;
    --green: #7A8A6C;
    --bg-offwhite: #FAF9F6;
    --panel-bg: rgba(255, 255, 255, 0.92);
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }

  body {
    font-family: 'Nunito', sans-serif;
    color: #333;
    overflow-x: hidden;
    min-height: 100vh;
  }

  /* FONDO GEEN.WEBP */
  .bg-image {
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    background-image: url('/geen.webp'); /* Asegúrate que esté en public */
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    z-index: -2;
    opacity: 0.6; /* Un poco transparente para que no moleste */
  }

  .app-wrapper {
    position: relative;
    min-height: 100vh;
  }

  /* CAPA DE BURBUJAS (Detrás de todo) */
  .bubble-layer {
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    pointer-events: none;
    z-index: 0; /* DETRÁS del contenido */
    overflow: hidden;
  }

  .bubble {
    position: absolute;
    bottom: -100px;
    left: var(--left);
    width: var(--size);
    height: var(--size);
    background: radial-gradient(circle at 30% 30%, rgba(255,255,255,0.9), var(--color));
    border-radius: 50%;
    opacity: 0.6;
    animation: floatUp var(--dur) linear infinite;
    animation-delay: var(--delay);
    filter: blur(1px);
  }

  @keyframes floatUp {
    0% { transform: translateY(0); opacity: 0; }
    10% { opacity: 0.6; }
    100% { transform: translateY(-120vh); opacity: 0; }
  }

  .content {
    position: relative;
    z-index: 10; /* ENCIMA de las burbujas */
    max-width: 600px;
    margin: 0 auto;
    padding: 40px 20px;
  }

  .retro-header {
    text-align: center;
    margin-bottom: 30px;
    text-shadow: 2px 2px 0px white;
  }

  .logo {
    font-family: 'VT323', monospace;
    font-size: 3.5rem;
    color: var(--pink);
    letter-spacing: 2px;
    line-height: 1;
  }

  .subtitle {
    font-family: 'VT323', monospace;
    font-size: 1.2rem;
    color: var(--green);
    text-transform: uppercase;
  }

  /* ESTRUCTURA DE CUADRO (PANEL) */
  .main-panel {
    background: var(--panel-bg);
    border: 4px double var(--pink); /* Borde doble Sweet-pea */
    padding: 30px;
    box-shadow: 8px 8px 0px rgba(212, 165, 165, 0.2);
    backdrop-filter: blur(5px);
  }

  /* INPUT RETRO (Estilo Ventana) */
  .input-window {
    border: 1px solid #999;
    background: #ececec;
    margin-bottom: 40px;
    box-shadow: 2px 2px 0px rgba(0,0,0,0.1);
  }

  .window-title-bar {
    background: var(--green);
    color: white;
    padding: 4px 8px;
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .window-controls div {
    display: inline-block;
    width: 10px; height: 10px;
    background: white;
    margin-left: 4px;
    border: 1px solid #999;
  }

  .window-content {
    padding: 12px;
  }

  .retro-input-name {
    width: 100%;
    border: none;
    border-bottom: 1px dashed var(--blue);
    background: transparent;
    font-family: 'VT323', monospace;
    font-size: 1.2rem;
    color: var(--blue);
    padding: 4px 0;
    margin-bottom: 8px;
    outline: none;
  }

  .retro-textarea {
    width: 100%;
    background: #fff;
    border: 1px solid var(--green); /* Borde verde sólido */
    border-radius: 0; /* Sin bordes redondos */
    padding: 10px;
    font-family: 'Nunito', sans-serif;
    font-size: 1rem;
    resize: vertical;
    outline: none;
    color: #333;
    box-shadow: inset 2px 2px 0px rgba(0,0,0,0.05);
  }

  .retro-textarea:focus {
    background: #fffff0;
  }

  .input-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 10px;
  }

  .mood-selector-mini {
    display: flex;
    gap: 6px;
  }

  .mood-dot {
    width: 20px; height: 20px;
    border-radius: 0; /* Cuadrados */
    cursor: pointer;
    border: 1px solid #ccc;
    transition: transform 0.2s;
  }

  .mood-dot.active {
    transform: scale(1.2);
    border: 2px solid var(--green);
  }

  .actions { display: flex; gap: 8px; }

  .file-input-hidden { display: none; }

  .btn-retro-attach, .btn-retro-send {
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    padding: 4px 12px;
    border: 1px solid #999;
    background: #ddd;
    cursor: pointer;
    box-shadow: 2px 2px 0px #999;
    active: translate(2px, 2px);
  }

  .btn-retro-send {
    background: var(--pink);
    color: white;
    border-color: #b08585;
  }

  .btn-retro-send:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .mini-preview {
    margin-top: 8px;
    position: relative;
    display: inline-block;
    border: 1px solid #999;
  }
  .mini-preview img { max-height: 60px; display: block; }
  .mini-preview button {
    position: absolute; top: -5px; right: -5px;
    background: red; color: white; border: none;
    width: 16px; height: 16px; font-size: 10px;
    cursor: pointer;
  }

  /* FEED */
  .feed-container {
    display: flex;
    flex-direction: column;
  }

  .loader-text, .empty-msg {
    text-align: center;
    font-family: 'VT323', monospace;
    font-size: 1.2rem;
    color: var(--text-light);
    padding: 20px;
  }
</style>