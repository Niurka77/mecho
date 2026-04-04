<script>
  export let post;

  function formatTime(dateString) {
    const d = new Date(dateString);
    return d.toLocaleString('es-ES', { 
      hour: '2-digit', minute: '2-digit', day: 'numeric', month: 'short' 
    });
  }
</script>

<article class="post-card" style="background-color: {post.color || '#FFFFFF'}">
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
    border-radius: 25px 30px 20px 35px;
    padding: 18px 20px;
    box-shadow: 0 6px 20px rgba(139, 154, 124, 0.12);
    display: flex;
    flex-direction: column;
    gap: 12px;
    transition: transform 0.3s ease, box-shadow 0.3s ease;
    border: 2px solid rgba(232, 213, 183, 0.5);
  }
  .post-card:hover { 
    transform: translateY(-4px) scale(1.01); 
    box-shadow: 0 12px 32px rgba(139, 154, 124, 0.2); 
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
    box-shadow: 0 6px 16px rgba(0,0,0,0.1);
    border: 2px solid rgba(255,255,255,0.6);
  }
  
  .post-audio {
    width: 100%;
    border-radius: 24px;
    background: rgba(249, 247, 243, 0.8);
    border: 2px solid rgba(232, 213, 183, 0.4);
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