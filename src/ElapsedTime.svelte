<script lang="ts">
  
  let { startTime } = $props<{ startTime: Date | number }>();

  let currentTime = $state(new Date());

  $effect(() => {
    const timer = setInterval(() => {
      currentTime = new Date();
    }, 1000);
    return () => clearInterval(timer);
  });

  let elapsedString = $derived.by(() => {
    if (!startTime) return '';
    const diffMs = currentTime.getTime() - new Date(startTime).getTime();
    const diffSecs = Math.max(0, Math.floor(diffMs / 1000));
    const mins = Math.floor(diffSecs / 60);
    const secs = diffSecs % 60;
    return `${mins}m ${secs < 10 ? '0' : ''}${secs}s`;
  });
</script>

<span>{elapsedString}</span>