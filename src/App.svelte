<script>
  import { onMount, onDestroy } from 'svelte';
  import { fly } from 'svelte/transition';
  import { supabase } from './lib/supabaseClient';
  import Post from './lib/Post.svelte';

  let posts = [];
  let loading = true;
  let textInput = '';
  let authorName = '';
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
  
  const moods = [
    { label: 'Normal', color: '#FFFFFF' },
    { label: 'Suave', color: '#FFF0F5' },
    { label: 'Cielo', color: '#F0F8FF' },
    { label: 'Menta', color: '#F0FFF0' }
  ];

  let selectedMood = moods[0];

  onMount(async () => {
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    await loadPosts();
    
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

  async function loadPosts() {
    try {
      const { data, error } = await supabase
        .from('posts')
        .select('*')
        .order('created_at', { ascending: false });
      
      if (error) throw error;
      posts = data || [];
      loading = false;
      updateTodayCount();
    } catch (err) {
      console.error('Error cargando posts:', err);
      loading = false;
    }
  }

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

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('Escribe algo o adjunta un archivo');
      return;
    }
    if (!authorName.trim()) {
      alert('Por favor, escribe tu nombre');
      return;
    }

    isUploading = true;
    let currentMediaUrl = null;

    try {
      if (mediaFile) {
        const fileExt = mediaFile.name.split('.').pop();
        const safeName = `${Date.now()}_${Math.random().toString(36).slice(2, 8)}.${fileExt}`;
        const datePath = new Date().toISOString().split('T')[0];
        const filePath = `posts/${datePath}/${safeName}`;

        const { error: uploadError } = await supabase.storage
          .from('mecho-media')
          .upload(filePath, mediaFile, { cacheControl: '3600' });

        if (uploadError) throw uploadError;
        
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
        author_name: authorName.trim(),
        media_url: currentMediaUrl,
        mood_color: selectedMood.color,
        type: postType,
        likes: 0
      };

      const { error: dbError } = await supabase
        .from('posts')
        .insert([newPostData]);
      
      if (dbError) throw dbError;

      // Limpiar formulario
      textInput = ''; 
      authorName = '';
      mediaFile = null; 
      mediaPreview = null;
      selectedMood = moods[0];
      if (fileInputRef) fileInputRef.value = '';
      
    } catch (error) {
      console.error('Error detallado:', error);
      alert('Error al guardar: ' + error.message);
    } finally {
      isUploading = false;
    }
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

  const bubbles = Array.from({ length: 8 }).map((_, i) => ({
    left: Math.random() * 90 + '%',
    delay: Math.random() * 5 + 's',
    duration: 10 + Math.random() * 10 + 's',
    size: 40 + Math.random() * 60 + 'px',
    color: ['#ffb3c1', '#b3e0f2', '#d4c1ec'][Math.floor(Math.random() * 3)]
  }));
</script>

<div class="app-wrapper">
  <div class="bg-image"></div>

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
      <!-- Mecho Mascot -->
    <div class="mascot-container">
      <img src="/mecho.png" alt="Mecho" class="mascot" />
      <div class="mascot-tooltip">
        {#if todayCount > 0}
          ¡{todayCount} registro{todayCount > 1 ? 's' : ''} hoy! 💖
        {:else}
          ¡Cuéntame tu día! 🌸
        {/if}
      </div>
    </div>

    <section class="main-panel">
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
                  type="button"
                ></button>
              {/each}
            </div>

            <div class="actions">
              <input 
                type="file" 
                accept="image/*,video/*,audio/*" 
                bind:this={fileInputRef} 
                on:change={handleFileChange} 
                class="file-input-hidden"
                id="media-input"
              />
              <label for="media-input" class="btn-retro-attach" title="Adjuntar archivo">
                📎
              </label>
              <button 
                class="btn-retro-send" 
                on:click={sendPost} 
                disabled={(!textInput.trim() && !mediaFile) || isUploading}
                type="button"
              >
                {#if isUploading}...{:else}ENVIAR{/if}
              </button>
            </div>
          </div>
          
          {#if mediaPreview}
            <div class="mini-preview">
              {#if mediaFile?.type?.startsWith('image')}
                <img src={mediaPreview} alt="vista previa" />
              {:else if mediaFile?.type?.startsWith('video')}
                <video src={mediaPreview} muted />
              {:else}
                <span> {mediaFile.name.slice(0, 20)}...</span>
              {/if}
              <button on:click={() => { mediaPreview=null; mediaFile=null; if(fileInputRef) fileInputRef.value=''; }} type="button">x</button>
            </div>
          {/if}
        </div>
      </div>

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

  .bg-image {
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    background-image: url('/geen.webp');
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    z-index: -2;
    opacity: 0.6;
  }

  .app-wrapper {
    position: relative;
    min-height: 100vh;
  }

  .bubble-layer {
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    pointer-events: none;
    z-index: 0;
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
    z-index: 10;
    max-width: 600px;
    margin: 0 auto;
    padding: 40px 20px 120px;
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

  .main-panel {
    background: var(--panel-bg);
    border: 4px double var(--pink);
    padding: 30px;
    box-shadow: 8px 8px 0px rgba(212, 165, 165, 0.2);
    backdrop-filter: blur(5px);
  }

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

  .retro-input-name::placeholder {
    color: #aaa;
  }

  .retro-textarea {
    width: 100%;
    background: #fff;
    border: 1px solid var(--green);
    border-radius: 0;
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

  .retro-textarea::placeholder {
    color: #999;
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
    border-radius: 0;
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
  }

  .btn-retro-attach:hover {
    background: #ccc;
  }

  .btn-retro-send {
    background: var(--pink);
    color: white;
    border-color: #b08585;
  }

  .btn-retro-send:hover:not(:disabled) {
    background: #c49595;
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
    background: white;
    padding: 4px;
  }
  
  .mini-preview img, .mini-preview video { 
    max-height: 60px; 
    display: block; 
    max-width: 200px;
  }
  
  .mini-preview button {
    position: absolute; 
    top: -5px; 
    right: -5px;
    background: red; 
    color: white; 
    border: none;
    width: 16px; 
    height: 16px; 
    font-size: 10px;
    cursor: pointer;
    line-height: 1;
  }

  .feed-container {
    display: flex;
    flex-direction: column;
  }

  .loader-text, .empty-msg {
    text-align: center;
    font-family: 'VT323', monospace;
    font-size: 1.2rem;
    color: #888;
    padding: 20px;
  }

/* Mecho Mascot */
.mascot-container {
  text-align: center;
  margin: 20px auto 30px;
  cursor: pointer;
  position: relative; /* Cambiado de fixed a relative */
}

  .mascot {
    width: 80px;
    height: auto;
    filter: drop-shadow(0 4px 8px rgba(0,0,0,0.2));
    transition: transform 0.3s;
    animation: float 3s ease-in-out infinite;
  }

  .mascot:hover {
    transform: scale(1.1);
  }

  .mascot:hover + .mascot-tooltip {
    opacity: 1;
     transform: translateY(0) translateX(-50%);  /* Agrega translateX */
  }

  .mascot-tooltip {
    position: absolute;
    bottom: 85px;
    right: 0;
    background: white;
    padding: 8px 12px;
    border-radius: 20px;
    font-size: 0.85rem;
    font-weight: 600;
    color: #333;
    left: 50%;      /* Agrega esto */
    white-space: nowrap;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    opacity: 0;
    transform: translateY(10px);
    transition: all 0.3s ease;
    pointer-events: none;
    border: 2px solid var(--pink);
  }

  @keyframes float {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-10px); }
  }

  @media (max-width: 600px) {
    .logo { font-size: 2.5rem; }
    .main-panel { padding: 20px; }
    .input-footer { flex-direction: column; gap: 10px; align-items: stretch; }
    .actions { justify-content: space-between; }
    .mascot { width: 60px; }
  }
</style>