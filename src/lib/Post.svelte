<script>
  export let post;

  function formatTime(dateString) {
    const d = new Date(dateString);
    return d.toLocaleString('es-ES', { 
      hour: '2-digit', minute: '2-digit', day: 'numeric', month: 'short' 
    });
  }
</script>

<article class="post-card">
  <!-- Contenido mixto: texto + media juntos -->
  {#if post.text}
    <p class="post-text">{post.text}</p>
  {/if}
  
  {#if post.media_url}
    {#if post.type === 'image'}
      <img src={post.media_url} alt="Recuerdo compartido" loading="lazy" class="post-media" />
    {:else if post.type === 'video'}
      <video controls src={post.media_url} class="post-media" preload="metadata"></video>
    {:else if post.type === 'audio'}
      <audio controls src={post.media_url} class="post-audio"></audio>
    {/if}
  {/if}

  <time class="post-time">{formatTime(post.created_at)}</time>
</article>

<style>
  .post-card {
    background: #FFFFFF;
    border-radius: 28px;
    padding: 18px 20px;
    box-shadow: 0 6px 20px rgba(139, 154, 124, 0.12);
    display: flex;
    flex-direction: column;
    gap: 12px;
    transition: transform 0.2s ease, box-shadow 0.2s ease;
    border: 1px solid rgba(232, 213, 183, 0.4);
  }
  .post-card:hover { 
    transform: translateY(-3px); 
    box-shadow: 0 10px 28px rgba(139, 154, 124, 0.18); 
  }
  
  .post-text {
    font-size: 1.05rem;
    line-height: 1.65;
    color: #4A4A4A;
    white-space: pre-wrap;
    word-wrap: break-word;
  }
  
  .post-media {
    width: 100%;
    border-radius: 20px;
    object-fit: cover;
    max-height: 420px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  }
  
  .post-audio {
    width: 100%;
    border-radius: 24px;
    background: #F9F7F3;
  }
  
  .post-time {
    font-size: 0.75rem;
    color: #8B9A7C;
    text-align: right;
    font-family: 'VT323', monospace;
    letter-spacing: 0.5px;
    margin-top: 4px;
    opacity: 0.9;
  }
</style>