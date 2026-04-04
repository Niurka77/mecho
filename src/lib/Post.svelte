<script>
  export let post;

  function formatTime(dateString) {
    const d = new Date(dateString);
    return d.toLocaleString('es-ES', { 
      hour: '2-digit', minute: '2-digit', day: 'numeric', month: 'short' 
    });
  }
</script>

<div class="post-card">
  {#if post.type === 'note'}
    <div class="note-content">{post.text}</div>
  {:else if post.type === 'image' && post.media_url}
    <img src={post.media_url} alt="Momento compartido" loading="lazy" />
  {:else if post.type === 'audio' && post.media_url}
    <audio controls src={post.media_url} class="media-player"></audio>
  {:else if post.type === 'video' && post.media_url}
    <video controls src={post.media_url} class="media-player"></video>
  {/if}
  
  <span class="timestamp">{formatTime(post.created_at)}</span>
</div>

<style>
  .post-card {
    background: #FFFFFF;
    border-radius: 28px;
    padding: 18px;
    margin-bottom: 18px;
    box-shadow: 0 6px 20px rgba(139, 154, 124, 0.12);
    display: flex;
    flex-direction: column;
    gap: 10px;
    transition: transform 0.2s ease, box-shadow 0.2s ease;
  }
  .post-card:hover { transform: translateY(-3px); box-shadow: 0 8px 24px rgba(139, 154, 124, 0.18); }
  img, video { width: 100%; border-radius: 20px; object-fit: cover; max-height: 420px; }
  audio { width: 100%; border-radius: 24px; }
  .note-content {
    background: #E8D5B7;
    padding: 16px;
    border-radius: 18px;
    font-size: 1.05rem;
    line-height: 1.65;
    color: #4A4A4A;
  }
  .timestamp {
    font-size: 0.75rem;
    color: #8B9A7C;
    text-align: right;
    font-family: 'VT323', monospace;
    letter-spacing: 0.5px;
  }
</style>