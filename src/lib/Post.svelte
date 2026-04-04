<script>
  export let post;
  export let onLike = () => {};
  
  let liked = false;
  let localLikes = post.likes || 0;
  
  function formatTime(dateString) {
    const d = new Date(dateString);
    return d.toLocaleString('es-ES', { 
      month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit'
    }).replace('.', '');
  }
  
  function handleLike() {
    liked = !liked;
    localLikes = localLikes + (liked ? 1 : -1);
    onLike(post.id, liked);
  }
</script>

<article class="guestbook-entry" style="background-color: {post.mood_color || '#FFFFFF'}">
  
  <!-- Cabecera: Fecha y Autor (Estilo Retro) -->
  <div class="entry-header">
    <span class="meta-date">{formatTime(post.created_at)}</span>
    <span class="meta-author">— {post.author_name || 'anónimo'}</span>
  </div>

  <!-- Contenido -->
  <div class="entry-body">
    {#if post.text}
      <p class="entry-text">{post.text}</p>
    {/if}
    
    {#if post.media_url}
      <div class="entry-media">
        {#if post.type === 'image'}
          <img src={post.media_url} alt="adjunto" loading="lazy" />
        {:else if post.type === 'video'}
          <video controls src={post.media_url}></video>
        {/if}
      </div>
    {/if}
  </div>

  <!-- Pie: Likes simples -->
  <div class="entry-footer">
    <button class="like-trigger {liked ? 'active' : ''}" on:click|preventDefault={handleLike}>
      [{localLikes}]
    </button>
  </div>

</article>

<style>
  @import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');

  .guestbook-entry {
    /* Fondo ligeramente más claro que el panel para notar el recuadro */
    background-color: #FFFCF8; 
    padding: 15px 10px;
    position: relative;
    transition: background-color 0.3s;
  }

  /* LÍNEA HORIZONTAL FINA Y PUNTEADA (Regla estricta) */
  .guestbook-entry + .guestbook-entry {
    border-top: 1px dashed var(--blue, #9DB5C7);
  }
  
  /* Si es el primer elemento, quizás quieras un borde arriba también o no, 
     dependiendo si hay un título. Aquí lo dejo limpio. */

  .entry-header {
    font-family: 'VT323', monospace;
    font-size: 1rem;
    color: #888;
    margin-bottom: 8px;
    display: flex;
    justify-content: space-between;
    text-transform: lowercase;
  }

  .meta-author {
    color: var(--green, #7A8A6C);
    font-weight: bold;
  }

  .entry-text {
    font-family: 'Nunito', sans-serif;
    font-size: 1rem;
    line-height: 1.5;
    color: #333;
    white-space: pre-wrap;
    margin-bottom: 10px;
  }

  .entry-media {
    margin-top: 10px;
    border: 1px solid #eee;
    padding: 4px;
    background: white;
    display: inline-block;
    max-width: 100%;
  }

  .entry-media img, .entry-media video {
    max-width: 100%;
    max-height: 300px;
    display: block;
  }

  .entry-footer {
    margin-top: 8px;
    text-align: right;
  }

  .like-trigger {
    font-family: 'VT323', monospace;
    background: none;
    border: none;
    color: #aaa;
    cursor: pointer;
    font-size: 1.1rem;
  }

  .like-trigger:hover, .like-trigger.active {
    color: var(--pink, #D4A5A5);
  }
</style>