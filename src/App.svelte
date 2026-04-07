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
  let isUploading = false;
  let uploadProgress = 0;
  let todayCount = 0;
  let popSound;
  
  // === 🟢 SISTEMA DE IDENTIDAD INVISIBLE ===
  let currentUser = "";
  let showNameSetup = false;
  let tempNameInput = '';
  let deviceId = ''; // Huella digital del navegador
  
  const MAX_CHARS = 500;
  
  const moods = [
    { label: 'Normal', color: '#FFFFFF' },
    { label: 'Suave', color: '#FFF0F5' },
    { label: 'Cielo', color: '#F0F8FF' },
    { label: 'Menta', color: '#F0FFF0' }
  ];

  let selectedMood = moods[0];
  let searchTerm = '';
  let editingPost = null;
  let originalTextInput = '';
  
  // === PAGINACIÓN ===
  let currentPage = 1;
  const postsPerPage = 5;
  
  $: filteredPosts = posts.filter(post => 
    (post.text?.toLowerCase() || '').includes(searchTerm.toLowerCase()) ||
    (post.author_name?.toLowerCase() || '').includes(searchTerm.toLowerCase())
  );
  
  $: paginatedPosts = filteredPosts.slice(
    (currentPage - 1) * postsPerPage, 
    currentPage * postsPerPage
  );
  
  $: totalPages = Math.ceil(filteredPosts.length / postsPerPage);
  
  function goToPage(page) {
    if (page >= 1 && page <= totalPages) {
      currentPage = page;
      document.querySelector('.feed-container')?.scrollIntoView({ behavior: 'smooth' });
    }
  }
  
  function handleNewPost(payload) {
    posts = [payload.new, ...posts];
    updateTodayCount();
    currentPage = 1;
    if (popSound) popSound.play().catch(() => {});
  }

  onMount(async () => {
    popSound = new Audio('/soft-pop.mp3');
    popSound.volume = 0.15;
    
    // 1. Generar o recuperar la Huella Digital (Invisible)
    deviceId = localStorage.getItem('mecho_device_id');
    if (!deviceId) {
      deviceId = crypto.randomUUID(); // Genera ID único
      localStorage.setItem('mecho_device_id', deviceId);
    }

    // 2. Verificar si ya tiene nombre guardado
    const savedName = localStorage.getItem('mecho_user');
    if (savedName && savedName.trim() !== '') {
      currentUser = savedName;
      showNameSetup = false;
    } else {
      showNameSetup = true;
    }
    
    await loadPosts();
    
    channel = supabase
      .channel('mecho-realtime')
      .on('postgres_changes', 
        { event: 'INSERT', schema: 'public', table: 'posts' }, 
        payload => handleNewPost(payload)
      )
      .subscribe();

    window.addEventListener('scroll', handleScroll);
    
    return () => {
      window.removeEventListener('scroll', handleScroll);
    };
  });

  // 🟢 GUARDAR NOMBRE
  function saveUserName(nombre) {
    const nombreLimpio = nombre.trim().slice(0, 20);
    if (nombreLimpio) {
      localStorage.setItem('mecho_user', nombreLimpio);
      currentUser = nombreLimpio;
      showNameSetup = false;
      tempNameInput = '';
    }
  }

  // 🟢 CAMBIAR NOMBRE
  function changeUserName() {
    tempNameInput = currentUser;
    showNameSetup = true;
  }

  // 🟢 CERRAR SESIÓN
  function logout() {
    if (confirm('¿Quieres cambiar de nombre?')) {
      localStorage.removeItem('mecho_user');
      currentUser = "";
      showNameSetup = true;
    }
  }

  // 🛡️ SEGURIDAD: Solo permite editar si el nombre Y la huella digital coinciden
  function puedeModificar(post) {
    return currentUser && 
           post.author_name === currentUser && 
           post.device_id === deviceId; 
  }
  
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
    todayCount = filteredPosts.filter(p => new Date(p.created_at).toDateString() === today).length;
  }

  function handleFileChange(event) {
    const file = event.target.files[0];
    if (!file) {
      mediaFile = null;
      mediaPreview = null;
      return;
    }
    
    if (file.size > 5 * 1024 * 1024) {
      alert('⚠️ El archivo es muy grande. Máximo 5MB');
      event.target.value = '';
      return;
    }
    
    const validTypes = [
      'image/jpeg', 'image/png', 'image/gif', 'image/webp',
      'video/mp4', 'video/webm',
      'audio/mpeg', 'audio/mp3', 'audio/wav'
    ];
    if (!validTypes.includes(file.type)) {
      alert('⚠️ Tipo de archivo no permitido. Solo imágenes, videos o audio.');
      event.target.value = '';
      return;
    }
    
    mediaFile = file;
    const reader = new FileReader();
    reader.onload = e => { mediaPreview = e.target.result; };
    reader.readAsDataURL(mediaFile);
  }

  async function deletePost(postId) {
    if (!confirm('¿Eliminar este mensaje permanentemente?')) return;
    
    try {
      const { error } = await supabase
        .from('posts')
        .delete()
        .eq('id', postId);
      
      if (error) throw error;
      posts = posts.filter(p => p.id !== postId);
    } catch (err) {
      alert('❌ Error al eliminar: ' + err.message);
    }
  }

  function startEdit(post) {
    editingPost = post;
    originalTextInput = textInput;
    textInput = post.text || '';
    textareaRef?.focus();
  }

  async function saveEdit() {
    if (!textInput.trim() || !editingPost) return;
    
    try {
      const { error } = await supabase
        .from('posts')
        .update({ text: textInput })
        .eq('id', editingPost.id);
      
      if (error) throw error;
      
      const index = posts.findIndex(p => p.id === editingPost.id);
      if (index !== -1) {
        posts[index].text = textInput;
        posts = [...posts];
      }
      cancelEdit();
    } catch (err) {
      alert('❌ Error al editar: ' + err.message);
    }
  }

  function cancelEdit() {
    editingPost = null;
    textInput = originalTextInput;
    originalTextInput = '';
  }

  async function sendPost() {
    if (!textInput.trim() && !mediaFile) {
      alert('✍️ Escribe algo o adjunta un archivo');
      return;
    }
    
    if (!currentUser) {
      alert('👤 Por favor, escribe tu nombre primero');
      showNameSetup = true;
      return;
    }

    isUploading = true;
    uploadProgress = 0;
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
        author_name: currentUser,
        media_url: currentMediaUrl,
        mood_color: selectedMood.color,
        type: postType,
        likes: 0,
        device_id: deviceId // 🛡️ GUARDAMOS LA HUELLA DIGITAL
      };

      const { error: dbError } = await supabase
        .from('posts')
        .insert([newPostData]);
      
      if (dbError) throw dbError;

      textInput = ''; 
      mediaFile = null; 
      mediaPreview = null;
      selectedMood = moods[0];
      if (fileInputRef) fileInputRef.value = '';
      
    } catch (error) {
      console.error('Error:', error);
      alert('❌ Error al guardar: ' + error.message);
    } finally {
      isUploading = false;
      uploadProgress = 0;
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

{#if showNameSetup}
  <div class="setup-overlay">
    <div class="setup-modal" transition:fly={{ y: 20, duration: 300 }}>
      <h2>👋 ¡Hola!</h2>
      <p>{currentUser ? 'Cambia tu nombre:' : 'Elige tu nombre para firmar tus mensajes:'}</p>
      <div class="name-input-group">
        <input 
          type="text" 
          bind:value={tempNameInput}
          placeholder="Ej: Seli, Dayna, Mecho..." 
          class="retro-input-name setup-input"
          maxlength="20"
          on:keydown={(e) => e.key === 'Enter' && tempNameInput.trim() && saveUserName(tempNameInput)}
          autofocus
        />
        <button 
          class="btn-retro-send" 
          on:click={() => tempNameInput.trim() && saveUserName(tempNameInput)}
          disabled={!tempNameInput.trim()}
          type="button"
        >
          {currentUser ? 'Actualizar' : '¡Listo!'} ✨
        </button>
      </div>
      {#if currentUser}
        <button class="btn-cancel-setup" on:click={() => showNameSetup = false} type="button">
          Cancelar
        </button>
      {/if}
    </div>
  </div>
{/if}

  <main class="content">
    <header class="retro-header">
      <h1 class="logo">GUESTBOOK</h1>
      <p class="subtitle">deja tu huella suave</p>
    </header>

    <div class="mascot-container">
      <img src="/mecho.png" alt="Mecho" class="mascot" />
      <div class="mascot-tooltip">
        {#if currentUser}
          <div>¡Hola {currentUser}! 💖</div>
          {#if todayCount > 0}
            <div>{todayCount} registro{todayCount > 1 ? 's' : ''} hoy</div>
          {/if}
        {:else}
          ¡Cuéntame tu día! 🌸
        {/if}
      </div>
    </div>

    <section class="main-panel">
      <div class="input-window">
        <div class="window-title-bar">
          <span>mensajitos</span>
          <div class="window-controls"><div></div><div></div></div>
        </div>
        
        <div class="window-content">
          {#if currentUser}
            <div class="current-user-display">
              <span class="user-label">👤</span>
              <span class="user-name">{currentUser}</span>
              <div class="user-actions">
                <button class="btn-change-name" on:click={changeUserName} type="button" title="Cambiar nombre">✏️</button>
                <button class="btn-logout" on:click={logout} type="button" title="Cerrar sesión">✕</button>
              </div>
            </div>
          {/if}
          
          <textarea 
            bind:value={textInput} 
            bind:this={textareaRef}
            placeholder={editingPost ? "editando mensaje..." : "escribe algo aquí.. :D"}
            rows="4"
            class="retro-textarea {editingPost ? 'editing-mode' : ''}"
            maxlength={MAX_CHARS}
          ></textarea>
          
          <div class="char-counter {textInput.length > MAX_CHARS * 0.9 ? 'warning' : ''}">
            {textInput.length}/{MAX_CHARS}
          </div>

          <div class="input-footer">
            <div class="mood-selector-mini">
              {#each moods as mood}
                <button 
                  class="mood-dot {selectedMood === mood ? 'active' : ''}"
                  style="background-color: {mood.color}; border: 1px solid #ccc"
                  on:click={() => selectedMood = mood}
                  title={mood.label}
                  type="button"
                  disabled={editingPost !== null}
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
                disabled={editingPost !== null || isUploading}
              />
              <label for="media-input" class="btn-retro-attach {editingPost !== null ? 'disabled' : ''}" title="Adjuntar archivo">
                📎
              </label>
              
              {#if editingPost}
                <button class="btn-retro-send btn-save" on:click={saveEdit} type="button">💾 Guardar</button>
                <button class="btn-retro-cancel" on:click={cancelEdit} type="button">✕ Cancelar</button>
              {:else}
                <button 
                  class="btn-retro-send" 
                  on:click={sendPost} 
                  disabled={(!textInput.trim() && !mediaFile) || isUploading || !currentUser}
                  type="button"
                >
                  {#if isUploading}
                    {#if uploadProgress > 0}
                      {uploadProgress}%
                    {:else}
                      ⏳
                    {/if}
                  {:else}
                    ENVIAR
                  {/if}
                </button>
              {/if}
            </div>
          </div>
          
          {#if isUploading && mediaFile}
            <div class="upload-progress">
              <div class="progress-bar" style="width: {uploadProgress || 30}%"></div>
            </div>
          {/if}
          
          {#if mediaPreview && !editingPost}
            <div class="mini-preview">
              {#if mediaFile?.type?.startsWith('image')}
                <img src={mediaPreview} alt="vista previa" />
              {:else if mediaFile?.type?.startsWith('video')}
                <video src={mediaPreview} muted>
                  <track kind="captions" />
                </video>
              {:else}
                <span>📄 {mediaFile.name.slice(0, 20)}...</span>
              {/if}
              <button on:click={() => { mediaPreview=null; mediaFile=null; if(fileInputRef) fileInputRef.value=''; }} type="button">✕</button>
            </div>
          {/if}
        </div>
      </div>

      <div class="search-bar">
        <input 
          type="search" 
          bind:value={searchTerm}
          placeholder="🔍 buscar mensajes o autores..."
          class="search-input"
        />
        {#if searchTerm}
          <button class="search-clear" on:click={() => searchTerm = ''} type="button">✕</button>
        {/if}
      </div>

      <section class="feed-container">
        {#if loading}
          <div class="loader-text">cargando memorias...</div>
        {:else if filteredPosts.length === 0}
          <div class="empty-msg">
            {#if searchTerm}
              🔍 No se encontraron resultados para "{searchTerm}"
            {:else}
              el libro está vacío.
            {/if}
          </div>
        {:else}
          {#each paginatedPosts as post (post.id)}
            <div transition:fly={{ y: 10, duration: 400 }}>
              <Post 
                {post} 
                onLike={handleLike} 
                onEdit={startEdit} 
                onDelete={deletePost}
                isEditing={editingPost?.id === post.id}
                currentUser={currentUser}
              />
            </div>
          {/each}
          
          {#if totalPages > 1}
            <div class="pagination-controls">
              <button 
                class="page-btn" 
                on:click={() => goToPage(currentPage - 1)} 
                disabled={currentPage === 1}
                type="button"
              >
                ◀ anterior
              </button>
              
              <div class="page-numbers">
                {#each Array.from({ length: totalPages }, (_, i) => i + 1) as pageNum}
                  <button 
                    class="page-number {pageNum === currentPage ? 'active' : ''}" 
                    on:click={() => goToPage(pageNum)}
                    type="button"
                  >
                    {pageNum}
                  </button>
                {/each}
              </div>
              
              <button 
                class="page-btn" 
                on:click={() => goToPage(currentPage + 1)} 
                disabled={currentPage === totalPages}
                type="button"
              >
                siguiente ▶
              </button>
            </div>
          {/if}
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
    --text: #333;
  }

  .setup-overlay {
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(0,0,0,0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    backdrop-filter: blur(4px);
  }

  .setup-modal {
    background: white;
    padding: 28px;
    border: 3px double var(--pink);
    border-radius: 12px;
    max-width: 340px;
    width: 90%;
    text-align: center;
    box-shadow: 8px 8px 0 rgba(212, 165, 165, 0.3);
  }

  .setup-modal h2 {
    font-family: 'VT323', monospace;
    color: var(--pink);
    margin-bottom: 12px;
    font-size: 2rem;
  }

  .setup-modal p {
    font-family: 'Nunito', sans-serif;
    margin-bottom: 16px;
    color: #555;
    font-size: 0.95rem;
  }

  .name-input-group {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 8px;
  }

  .setup-input {
    text-align: center;
    font-size: 1.3rem;
    padding: 10px;
    border: 2px solid var(--blue);
    border-radius: 6px;
    font-family: 'Nunito', sans-serif;
  }

  .btn-cancel-setup {
    margin-top: 12px;
    background: none;
    border: none;
    color: #999;
    cursor: pointer;
    font-family: 'Nunito', sans-serif;
    font-size: 0.9rem;
    text-decoration: underline;
  }
  .btn-cancel-setup:hover { color: var(--pink); }

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

  /* 🟢 NUEVO: Display de usuario actual */
  .current-user-display {
    display: flex;
    align-items: center;
    gap: 8px;
    background: rgba(157, 181, 199, 0.2);
    padding: 8px 12px;
    border-radius: 4px;
    margin-bottom: 12px;
    border: 1px dashed var(--blue);
  }

  .user-label {
    font-family: 'VT323', monospace;
    color: var(--green);
    font-size: 1rem;
  }

  .user-name {
    font-family: 'Nunito', sans-serif;
    font-weight: 600;
    color: var(--text);
    flex: 1;
  }

  .user-actions {
    display: flex;
    gap: 4px;
  }

  .btn-change-name, .btn-logout {
    background: none;
    border: none;
    color: #999;
    cursor: pointer;
    font-size: 1rem;
    padding: 2px 6px;
    border-radius: 4px;
    transition: all 0.2s;
  }

  .btn-change-name:hover {
    background: rgba(122, 138, 108, 0.2);
    color: var(--green);
  }
  
  .btn-logout:hover {
    background: rgba(212, 165, 165, 0.2);
    color: var(--pink);
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
    color: var(--text);
    box-shadow: inset 2px 2px 0px rgba(0,0,0,0.05);
  }

  .retro-textarea:focus { background: #fffff0; }
  .retro-textarea:disabled { background: #f5f5f5; opacity: 0.7; }
  .retro-textarea::placeholder { color: #999; }

  .char-counter {
    text-align: right;
    font-family: 'VT323', monospace;
    font-size: 0.9rem;
    color: #888;
    margin-top: 4px;
    transition: color 0.2s;
  }
  .char-counter.warning { color: var(--pink); font-weight: bold; }

  .input-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-top: 10px;
  }

  .mood-selector-mini { display: flex; gap: 6px; }

  .mood-dot {
    width: 20px; height: 20px;
    border-radius: 0;
    cursor: pointer;
    border: 1px solid #ccc;
    transition: transform 0.2s;
  }
  .mood-dot.active { transform: scale(1.2); border: 2px solid var(--green); }
  .mood-dot:disabled { opacity: 0.5; cursor: not-allowed; }

  .actions { display: flex; gap: 8px; }
  .file-input-hidden { display: none; }

  .btn-retro-attach, .btn-retro-send, .btn-retro-cancel {
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    padding: 4px 12px;
    border: 1px solid #999;
    background: #ddd;
    cursor: pointer;
    box-shadow: 2px 2px 0px #999;
  }
  .btn-retro-attach:hover { background: #ccc; }
  .btn-retro-attach.disabled { opacity: 0.5; cursor: not-allowed; }

  .btn-retro-send {
    background: var(--pink);
    color: white;
    border-color: #b08585;
  }
  .btn-retro-send:hover:not(:disabled) { background: #c49595; }
  .btn-retro-send:disabled { opacity: 0.6; cursor: not-allowed; }

  .btn-retro-cancel {
    background: #ccc;
    color: #333;
  }
  .btn-retro-cancel:hover { background: #bbb; }

  .upload-progress {
    margin-top: 10px;
    background: #eee;
    border-radius: 4px;
    overflow: hidden;
    position: relative;
    height: 20px;
    font-family: 'VT323', monospace;
    font-size: 0.9rem;
  }
  .progress-bar {
    height: 100%;
    background: linear-gradient(90deg, var(--pink), var(--blue));
    transition: width 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-weight: bold;
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

  .search-bar {
    position: relative;
    margin: 20px 0;
  }
  .search-input {
    width: 100%;
    padding: 10px 30px 10px 12px;
    border: 2px solid var(--blue);
    border-radius: 20px;
    font-family: 'Nunito', sans-serif;
    font-size: 1rem;
    background: white;
    outline: none;
    transition: border-color 0.2s;
  }
  .search-input,
  input[type="text"],
  input[type="search"],
  textarea {
    color: #1a1a1a !important;
    -webkit-text-fill-color: #1a1a1a !important;
  }

  .search-input::placeholder,
  input::placeholder,
  textarea::placeholder {
    color: #666 !important;
    opacity: 1 !important;
  }

  .search-input:focus { border-color: var(--pink); }
  .search-clear {
    position: absolute;
    right: 10px;
    top: 50%;
    transform: translateY(-50%);
    background: none;
    border: none;
    color: #999;
    cursor: pointer;
    font-size: 1.2rem;
    padding: 4px 8px;
  }
  .search-clear:hover { color: var(--pink); }

  .feed-container { display: flex; flex-direction: column; }

  .loader-text, .empty-msg {
    text-align: center;
    font-family: 'VT323', monospace;
    font-size: 1.2rem;
    color: #888;
    padding: 20px;
  }

  .mascot-container {
    text-align: center;
    margin: 20px auto 30px;
    cursor: pointer;
    position: relative;
  }
  .mascot {
    width: 80px;
    height: auto;
    filter: drop-shadow(0 4px 8px rgba(0,0,0,0.2));
    transition: transform 0.3s;
    animation: float 3s ease-in-out infinite;
  }
  .mascot:hover { transform: scale(1.1); }
  .mascot:hover + .mascot-tooltip {
    opacity: 1;
    transform: translateY(0) translateX(-50%);
  }
  .mascot-tooltip {
    position: absolute;
    bottom: 85px;
    left: 50%;
    background: white;
    padding: 8px 12px;
    border-radius: 20px;
    font-size: 0.85rem;
    font-weight: 600;
    color: #333;
    white-space: nowrap;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    opacity: 0;
    transform: translateY(10px) translateX(-50%);
    transition: all 0.3s ease;
    pointer-events: none;
    border: 2px solid var(--pink);
    text-align: center;
  }
  @keyframes float {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-10px); }
  }

  .pagination-controls {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 12px;
    margin-top: 24px;
    padding-top: 16px;
    border-top: 1px dashed var(--blue);
  }

  .page-btn {
    font-family: 'VT323', monospace;
    font-size: 1.1rem;
    padding: 6px 16px;
    background: white;
    border: 2px solid var(--pink);
    border-radius: 0;
    color: black;
    cursor: pointer;
    transition: all 0.2s;
    font-weight: bold;
  }

  .page-btn:hover:not(:disabled) { 
    background: var(--pink); 
    color: white; 
  }

  .page-btn:disabled { 
    opacity: 0.4; 
    cursor: not-allowed; 
  }

  .page-numbers { 
    display: flex; 
    gap: 6px; 
  }

  .page-number {
    font-family: 'VT323', monospace;
    font-size: 1rem;
    width: 28px;
    height: 28px;
    border: 1px solid var(--blue);
    background: white;
    color: black;
    cursor: pointer;
    transition: all 0.2s;
    font-weight: bold;
  }

  .page-number:hover {
    background: var(--blue);
    color: white;
  }

  .page-number.active {
    background: var(--green);
    color: white;
    border-color: var(--green);
  }

  .retro-textarea.editing-mode {
    border-color: var(--pink);
    background: #fffef0;
    box-shadow: 0 0 0 3px rgba(212, 165, 165, 0.2);
  }

  .btn-save {
    background: var(--green) !important;
    border-color: #5a6a4c !important;
  }
  .btn-save:hover {
    background: #6a7a5c !important;
  }

  @media (max-width: 600px) {
    .logo { font-size: 2.5rem; }
    .main-panel { padding: 20px; }
    .input-footer { flex-direction: column; gap: 10px; align-items: stretch; }
    .actions { justify-content: space-between; }
    .mascot { width: 60px; }
    .current-user-display {
      flex-wrap: wrap;
    }
  }
</style>