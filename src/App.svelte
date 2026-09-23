<script lang="ts">
  import ElapsedTime from './ElapsedTime.svelte';
 
  function formatTime(date: Date | string | number) {
    if (!date) return '';
    return new Date(date).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }

  let courts = $state([
    { id: 1, name: "Court 1", isAvailable: true, activeMatch: null as any },
    { id: 2, name: "Court 2", isAvailable: true, activeMatch: null as any },
    { id: 3, name: "Court 3", isAvailable: true, activeMatch: null as any }
  ]);

  // Sample players
  let players = $state([
    { id: 1, name: "Hawk Eye", level: "Low Advanced", registeredAt: new Date() },
    { id: 2, name: "Iron Man", level: "High Advanced", registeredAt: new Date() },
    { id: 3, name: "Mary Jane", level: "High Beginner", registeredAt: new Date() },
    { id: 4, name: "Gwen Stacey", level: "Low Intermediate", registeredAt: new Date() }
  ]);

  const levelWeights: Record<string, number> = {
    "High Advanced": 6, "Low Advanced": 5, "High Intermediate": 4,
    "Low Intermediate": 3, "High Beginner": 2, "Low Beginner": 1
  };
  let sortedPlayers = $derived([...players].sort((a, b) => levelWeights[b.level] - levelWeights[a.level]));

  // Grouped players by category rows for the Registered Players table
  let beginnerPlayers = $derived(sortedPlayers.filter(p => p.level.includes("Beginner")));
  let intermediatePlayers = $derived(sortedPlayers.filter(p => p.level.includes("Intermediate")));
  let advancedPlayers = $derived(sortedPlayers.filter(p => p.level.includes("Advanced")));

  // SECTION 3: The Waiting Queue (Queued Matches)
  let waitingQueue = $state([
    { id: 1, name: "Kiko & Marc vs Andrea & Miguel", type: "Doubles Match", p1Id: 1, p2Id: 2, p3Id: 3, p4Id: 4 }
  ]);

  // DERIVED: Find IDs of players currently locked inside the waiting queue or playing
  let queuedPlayerIds = $derived(
    new Set(waitingQueue.flatMap(m => [m.p1Id, m.p2Id, m.p3Id, m.p4Id]))
  );

  let activePlayerIds = $derived(
    new Set(courts.filter(c => !c.isAvailable && c.activeMatch).flatMap(c => [c.activeMatch.p1Id, c.activeMatch.p2Id, c.activeMatch.p3Id, c.activeMatch.p4Id]))
  );

  let busyPlayerIds = $derived(new Set([...queuedPlayerIds, ...activePlayerIds]));

  // DERIVED: Available players sorted by waiting time (Longest wait / oldest timestamp first)
  let availablePlayers = $derived(
    players
      .filter(player => !busyPlayerIds.has(player.id))
      .sort((a, b) => new Date(a.registeredAt).getTime() - new Date(b.registeredAt).getTime())
  );

  // Selected player IDs directly from the Available Players pool (max 4)
  let selectedPlayerIds = $state<number[]>([]);

  // Clear selections if any chosen player becomes busy
  $effect(() => {
    selectedPlayerIds = selectedPlayerIds.filter(id => availablePlayers.some(p => p.id === id));
  });

  function togglePlayerSelection(id: number) {
    if (selectedPlayerIds.includes(id)) {
      selectedPlayerIds = selectedPlayerIds.filter(pId => pId !== id);
    } else {
      if (selectedPlayerIds.length >= 4) {
        alert("You can only select up to 4 players for a doubles match!");
        return;
      }
      selectedPlayerIds = [...selectedPlayerIds, id];
    }
  }

  // SECTION 5: Match History
  let matchHistory = $state<any[]>([]);

  // Form states
  let newPlayerName = $state('');
  let newPlayerLevel = $state('Low Beginner');

  // Modals States
  let isEditModalOpen = $state(false);
  let editMatchId = $state<number | null>(null);
  let editP1Id, editP2Id, editP3Id, editP4Id = $state<number | string>('');

  let isPlayerModalOpen = $state(false);
  let editPlayerId = $state<number | null>(null);
  let editPlayerName = $state('');
  let editPlayerLevel = $state('');

  // Score Modal States
  let isScoreModalOpen = $state(false);
  let scoreCourtId = $state<number | null>(null);
  let scoreT1 = $state('');
  let scoreT2 = $state('');

  // --- Court Functions ---
  function addCourt() {
    let currentCount = courts.length > 0 ? Math.max(...courts.map(c => c.id)) : 0;
    courts = [...courts, { 
      id: currentCount + 1, 
      name: `Court ${currentCount + 1}`, 
      isAvailable: true, 
      activeMatch: null 
    }];
  }

  function deleteCourt(courtId: number) {
    courts = courts.filter(c => c.id !== courtId);
  }

  // --- Player Functions ---
  function addPlayer() {
    if (newPlayerName.trim() !== '') {
      players = [...players, { 
        id: Date.now(), 
        name: newPlayerName, 
        level: newPlayerLevel, 
        registeredAt: new Date() 
      }];
      newPlayerName = ''; 
    }
  }

  function openPlayerModal(player: any) {
    editPlayerId = player.id; editPlayerName = player.name; editPlayerLevel = player.level; isPlayerModalOpen = true;
  }
  function closePlayerModal() { isPlayerModalOpen = false; editPlayerId = null; }

  function saveEditedPlayer() {
    if (editPlayerName.trim() !== '' && editPlayerId !== null) {
      players = players.map(p => p.id === editPlayerId ? { ...p, name: editPlayerName, level: editPlayerLevel } : p);
      waitingQueue = waitingQueue.map(match => {
        if ([match.p1Id, match.p2Id, match.p3Id, match.p4Id].includes(editPlayerId as number)) {
          const getP = (id: number | string) => id === editPlayerId ? editPlayerName : players.find(p=>p.id===id)?.name;
          return { ...match, name: `${getP(match.p1Id)} & ${getP(match.p2Id)} vs ${getP(match.p3Id)} & ${getP(match.p4Id)}` };
        }
        return match;
      });
      closePlayerModal();
    }
  }
  function deletePlayer() {
    if (confirm("Are you sure you want to delete this player?")) { 
      if (busyPlayerIds.has(editPlayerId as number)) {
        alert("Cannot delete a player who is currently in a queued match or playing!");
        return;
      }
      players = players.filter(p => p.id !== editPlayerId); 
      closePlayerModal(); 
    }
  }

  // --- Queue Functions ---
  function queueSelectedMatch() {
    if (selectedPlayerIds.length !== 4) {
      alert("Please select exactly 4 players!");
      return;
    }
    const [p1Id, p2Id, p3Id, p4Id] = selectedPlayerIds;
    const p1 = players.find(p => p.id === p1Id);
    const p2 = players.find(p => p.id === p2Id);
    const p3 = players.find(p => p.id === p3Id);
    const p4 = players.find(p => p.id === p4Id);

    if (p1 && p2 && p3 && p4) {
      waitingQueue = [
        ...waitingQueue, 
        { 
          id: Date.now(), 
          name: `${p1.name} & ${p2.name} vs ${p3.name} & ${p4.name}`, 
          type: "Doubles Match", 
          p1Id, p2Id, p3Id, p4Id 
        }
      ];
      selectedPlayerIds = []; // Reset selection after queueing
    }
  }

  function startMatch(matchId: number) {
    const availableCourtIndex = courts.findIndex(c => c.isAvailable);
    if (availableCourtIndex === -1) return alert("All courts are currently full!");
    const matchToPlay = waitingQueue.find(m => m.id === matchId);
    if (matchToPlay) {
      courts[availableCourtIndex].activeMatch = { ...matchToPlay, startedAt: new Date() };
      courts[availableCourtIndex].isAvailable = false;
      waitingQueue = waitingQueue.filter(m => m.id !== matchId);
    }
  }

  function openEditModal(matchId: number) {
    const m = waitingQueue.find(m => m.id === matchId);
    if (m) { editMatchId = matchId; editP1Id = m.p1Id; editP2Id = m.p2Id; editP3Id = m.p3Id; editP4Id = m.p4Id; isEditModalOpen = true; }
  }
  function closeEditModal() { isEditModalOpen = false; editMatchId = null; }
  
  function saveEditedMatch() {
    if (editP1Id && editP2Id && editP3Id && editP4Id) {
      if (new Set([editP1Id, editP2Id, editP3Id, editP4Id]).size !== 4) return alert("Please ensure all 4 players are different!");
      const p1 = players.find(p => p.id === Number(editP1Id)); 
      const p2 = players.find(p => p.id === Number(editP2Id));
      const p3 = players.find(p => p.id === Number(editP3Id)); 
      const p4 = players.find(p => p.id === Number(editP4Id));
      if (p1 && p2 && p3 && p4) {
        waitingQueue = waitingQueue.map(match => {
          if (match.id === editMatchId) {
            return { ...match, name: `${p1.name} & ${p2.name} vs ${p3.name} & ${p4.name}`, p1Id: Number(editP1Id), p2Id: Number(editP2Id), p3Id: Number(editP3Id), p4Id: Number(editP4Id) };
          }
          return match;
        });
        closeEditModal();
      }
    }
  }
  function deleteQueuedMatch() { waitingQueue = waitingQueue.filter(m => m.id !== editMatchId); closeEditModal(); }

  // --- Score & History Functions ---
  function openScoreModal(courtId: number) {
    scoreCourtId = courtId; scoreT1 = ''; scoreT2 = ''; isScoreModalOpen = true;
  }
  function closeScoreModal() { isScoreModalOpen = false; scoreCourtId = null; }

  function finishMatchAndSave() {
    if (scoreCourtId === null) return;
    const courtIndex = courts.findIndex(c => c.id === scoreCourtId);
    const court = courts[courtIndex];

    if (court && court.activeMatch) {
      const match = court.activeMatch;

      matchHistory = [...matchHistory, {
        id: Date.now(),
        name: match.name,
        startedAt: match.startedAt,
        endedAt: new Date(),
        score: `${scoreT1} - ${scoreT2}`
      }];
      
      const finishedPlayerIds = [match.p1Id, match.p2Id, match.p3Id, match.p4Id];
      players = players.map(p => 
        finishedPlayerIds.includes(p.id) ? { ...p, registeredAt: new Date() } : p
      );

      court.activeMatch = null;
      court.isAvailable = true;
    }
    closeScoreModal();
  }
</script>

<main class="container">
  <h1>Badminton Queue Manager</h1>

  <!-- SECTION 1: Active Courts -->
  <section class="card">
    <div class="header-with-action">
      <h2>1. Active Courts</h2>
      <div class="add-court-controls">
        <button class="secondary-btn" onclick={addCourt}>+ Add Court</button>
      </div>
    </div>
    
    <div class="courts-grid">
      {#each courts as court}
        <div class="court {court.isAvailable ? 'available' : 'occupied'}">
          <div class="court-header">
            <input class="court-name-input" bind:value={court.name} />
            <div class="court-header-actions">
              <span class="status">{court.isAvailable ? 'Open' : 'In Use'}</span>
              <button class="delete-icon-btn" title="Delete Court" onclick={() => deleteCourt(court.id)}>✕</button>
            </div>
          </div>
          
          <div class="court-players">
            {#if court.isAvailable}
              <p class="empty-text">Waiting for players...</p>
            {:else}
              <p class="playing">{court.activeMatch.name}</p>
              <p class="time-started">Match Time: <ElapsedTime startTime={court.activeMatch.startedAt} /></p>
              <button class="clear-btn" onclick={() => openScoreModal(court.id)}>End Match & Score</button>
            {/if}
          </div>
        </div>
      {/each}
    </div>
  </section>

  <!-- SECTION 2: Registered Players Table -->
  <section class="card">
    <h2>2. Registered Players</h2>
    <div class="form-row">
      <input type="text" bind:value={newPlayerName} placeholder="Enter player name" />
      <select bind:value={newPlayerLevel}>
        <option>Low Beginner</option>
        <option>High Beginner</option>
        <option>Low Intermediate</option>
        <option>High Intermediate</option>
        <option>Low Advanced</option>
        <option>High Advanced</option>
      </select>
      <button class="primary-btn" onclick={addPlayer}>Add Player</button>
    </div>

    <div class="table-container">
      <table class="registered-table">
        <thead>
          <tr>
            <th style="width: 180px;">Tier</th>
            <th>Registered Players</th>
          </tr>
        </thead>
        <tbody>
          <!-- Advanced Row -->
          <tr>
            <td class="tier-cell advanced-tier">Advanced</td>
            <td>
              <div class="player-pool-table">
                {#if advancedPlayers.length === 0}
                  <span class="empty-row-text">No advanced players registered.</span>
                {:else}
                  {#each advancedPlayers as player}
                    <button class="player-chip" onclick={() => openPlayerModal(player)} title="Click to edit player">
                      {player.name} ({player.level})
                      <span class="chip-time">🕒 {formatTime(player.registeredAt)}</span>
                    </button>
                  {/each}
                {/if}
              </div>
            </td>
          </tr>

          <!-- Intermediate Row -->
          <tr>
            <td class="tier-cell intermediate-tier">Intermediate</td>
            <td>
              <div class="player-pool-table">
                {#if intermediatePlayers.length === 0}
                  <span class="empty-row-text">No intermediate players registered.</span>
                {:else}
                  {#each intermediatePlayers as player}
                    <button class="player-chip" onclick={() => openPlayerModal(player)} title="Click to edit player">
                      {player.name} ({player.level})
                      <span class="chip-time">🕒 {formatTime(player.registeredAt)}</span>
                    </button>
                  {/each}
                {/if}
              </div>
            </td>
          </tr>

          <!-- Beginner Row -->
          <tr>
            <td class="tier-cell beginner-tier">Beginner</td>
            <td>
              <div class="player-pool-table">
                {#if beginnerPlayers.length === 0}
                  <span class="empty-row-text">No beginner players registered.</span>
                {:else}
                  {#each beginnerPlayers as player}
                    <button class="player-chip" onclick={() => openPlayerModal(player)} title="Click to edit player">
                      {player.name} ({player.level})
                      <span class="chip-time">🕒 {formatTime(player.registeredAt)}</span>
                    </button>
                  {/each}
                {/if}
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <small class="empty-text">(Click a player to edit or delete)</small>
  </section>

  <!-- SECTION 3: Available Players & Direct Queue Selection -->
  <section class="card">
    <div class="header-with-action">
      <h2>3. Available Players</h2>
      <div class="selection-status-group">
        <span class="team-tag tag-t1">Team 1: {Math.min(selectedPlayerIds.length, 2)}/2</span>
        <span class="team-tag tag-t2">Team 2: {Math.max(0, selectedPlayerIds.length - 2)}/2</span>
      </div>
    </div>
    <p class="helper-hint">Click 2 players for <strong style="color: #3498db;">Team 1 (Blue)</strong> and 2 players for <strong style="color: #e74c3c;">Team 2 (Red)</strong></p>

    <div class="player-pool available-pool">
      {#if availablePlayers.length === 0}
        <p class="empty-text" style="margin: 0; width: 100%;">No available players waiting (all are queued or playing).</p>
      {:else}
        {#each availablePlayers as player}
          {@const index = selectedPlayerIds.indexOf(player.id)}
          {@const isTeam1 = index === 0 || index === 1}
          {@const isTeam2 = index === 2 || index === 3}
          <button 
            class="player-chip selectable-chip {isTeam1 ? 'selected-team1' : ''} {isTeam2 ? 'selected-team2' : ''}" 
            onclick={() => togglePlayerSelection(player.id)}
          >
            <span class="checkbox-indicator">
              {#if isTeam1}T1{:else if isTeam2}T2{:else}+{/if}
            </span>
            {player.name} ({player.level})
            <span class="chip-time">⏳ <ElapsedTime startTime={player.registeredAt} /></span>
          </button>
        {/each}
      {/if}
    </div>

    {#if selectedPlayerIds.length === 4}
      <button class="primary-btn action-btn full-width" onclick={queueSelectedMatch}>
        🚀 Queue Match (Team 1 vs Team 2)
      </button>
    {/if}
  </section>

  <!-- SECTION 4: Queued Matches -->
  <section class="card">
    <h2>4. Queued Matches ({waitingQueue.length})</h2>
    <div class="queue-list">
      {#each waitingQueue as match, index}
        <div class="queue-item">
          <div class="team-info">
            <span class="queue-number">#{index + 1}</span>
            <div>
              <div class="team-name">{match.name}</div>
              <span class="match-badge">{match.type}</span>
            </div>
          </div>
          <div class="queue-actions">
            <button class="edit-btn" onclick={() => openEditModal(match.id)}>Edit</button>
            <button class="play-btn" onclick={() => startMatch(match.id)}>Send to Court ▶</button>
          </div>
        </div>
      {/each}
    </div>
  </section>

  <!-- SECTION 5: Match History -->
  <section class="card">
    <h2>5. Match History</h2>
    {#if matchHistory.length === 0}
      <p class="empty-text">No matches have been completed yet.</p>
    {:else}
      <div class="history-list">
        {#each matchHistory.slice().reverse() as history}
          <div class="history-item">
            <div class="history-details">
              <div class="history-teams">{history.name}</div>
              <div class="history-times">🕒 {formatTime(history.startedAt)} - {formatTime(history.endedAt)}</div>
            </div>
            <div class="history-score-badge">{history.score}</div>
          </div>
        {/each}
      </div>
    {/if}
  </section>
</main>

<!-- Player Edit Modal -->
{#if isPlayerModalOpen}
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div class="modal-backdrop" onclick={closePlayerModal}></div>
  <div class="modal">
    <div class="modal-header">
      <h2>Edit Player</h2>
      <button class="close-icon-btn" onclick={closePlayerModal}>✕</button>
    </div>
    
    <div class="form-row modal-matchup">
      <input type="text" bind:value={editPlayerName} placeholder="Player name" />
      <select bind:value={editPlayerLevel}>
        <option>Low Beginner</option>
        <option>High Beginner</option>
        <option>Low Intermediate</option>
        <option>High Intermediate</option>
        <option>Low Advanced</option>
        <option>High Advanced</option>
      </select>
    </div>

    <div class="modal-footer">
      <button class="danger-btn" onclick={deletePlayer}>Delete Player</button>
      <div class="modal-right-actions">
        <button class="secondary-btn" onclick={closePlayerModal}>Cancel</button>
        <button class="primary-btn" onclick={saveEditedPlayer}>Save Changes</button>
      </div>
    </div>
  </div>
{/if}

<!-- Match Edit Modal -->
{#if isEditModalOpen}
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div class="modal-backdrop" onclick={closeEditModal}></div>
  <div class="modal">
    <div class="modal-header">
      <h2>Edit Match</h2>
      <button class="close-icon-btn" onclick={closeEditModal}>✕</button>
    </div>

    <div class="matchup-container modal-matchup">
      <div class="team-box">
        <h3>Team 1</h3>
        <select bind:value={editP1Id}>{#each sortedPlayers as player}<option value={player.id}>{player.name} ({player.level})</option>{/each}</select>
        <select bind:value={editP2Id}>{#each sortedPlayers as player}<option value={player.id}>{player.name} ({player.level})</option>{/each}</select>
      </div>
      <div class="vs-badge">VS</div>
      <div class="team-box">
        <h3>Team 2</h3>
        <select bind:value={editP3Id}>{#each sortedPlayers as player}<option value={player.id}>{player.name} ({player.level})</option>{/each}</select>
        <select bind:value={editP4Id}>{#each sortedPlayers as player}<option value={player.id}>{player.name} ({player.level})</option>{/each}</select>
      </div>
    </div>

    <div class="modal-footer">
      <button class="danger-btn" onclick={deleteQueuedMatch}>Delete Match</button>
      <div class="modal-right-actions">
        <button class="secondary-btn" onclick={closeEditModal}>Cancel</button>
        <button class="primary-btn" onclick={saveEditedMatch}>Save Changes</button>
      </div>
    </div>
  </div>
{/if}

<!-- End Match & Score Modal -->
{#if isScoreModalOpen}
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div class="modal-backdrop" onclick={closeScoreModal}></div>
  <div class="modal">
    <div class="modal-header">
      <h2>Enter Final Score</h2>
      <button class="close-icon-btn" onclick={closeScoreModal}>✕</button>
    </div>
    
    <div class="matchup-container modal-matchup">
      <div class="score-box">
        <label for="t1">Team 1 Score</label>
        <input type="number" id="t1" bind:value={scoreT1} min="0" class="score-input" />
      </div>
      <div class="vs-badge">VS</div>
      <div class="score-box">
        <label for="t2">Team 2 Score</label>
        <input type="number" id="t2" bind:value={scoreT2} min="0" class="score-input" />
      </div>
    </div>

    <div class="modal-footer">
      <button class="secondary-btn" onclick={closeScoreModal}>Cancel</button>
      <button class="primary-btn" onclick={finishMatchAndSave}>Save & End Match</button>
    </div>
  </div>
{/if}

<style>
  :global(body) { font-family: system-ui, sans-serif; background: #f4f7f6; margin: 0; padding: 2rem; color: #1473d3; }
  h1 { color: #2c3e50; }
  h2 { color: #34495e; margin-top: 0; }
  h3 { color: #2c3e50; margin: 0 0 0.5rem 0; font-size: 1rem; text-align: center; }

  .container { max-width: 900px; margin: 0 auto; position: relative; }
  .card { background: white; padding: 1.5rem; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); margin-bottom: 2rem; }

  .form-row { display: flex; align-items: center; gap: 1rem; margin-top: 1rem; }
  input, select { padding: 0.8rem; border: 1px solid #ccc; border-radius: 6px; font-size: 1rem; width: 100%; box-sizing: border-box; }
  
  .primary-btn { background: #27ae60; color: white; border: none; padding: 0.8rem 1.5rem; border-radius: 6px; font-weight: bold; cursor: pointer; white-space: nowrap; }
  .primary-btn:hover { background: #219a52; }
  .secondary-btn { background: #ecf0f1; color: #2c3e50; border: 1px solid #bdc3c7; padding: 0.8rem 1rem; border-radius: 6px; font-weight: bold; cursor: pointer; }
  .secondary-btn:hover { background: #dfe6e9; }
  .danger-btn { background: #e74c3c; color: white; border: none; padding: 0.8rem 1.5rem; border-radius: 6px; font-weight: bold; cursor: pointer; }
  .danger-btn:hover { background: #c0392b; }

  .action-btn { background: #27ae60; margin-top: 1.5rem; animation: pulse 1.5s infinite; }
  .action-btn:hover { background: #219a52; }
  .full-width { width: 100%; font-size: 1.1rem; }

  @keyframes pulse {
    0% { transform: scale(1); }
    50% { transform: scale(1.01); }
    100% { transform: scale(1); }
  }

  /* Registered Players Table Styles */
  .table-container { margin-top: 1.5rem; overflow-x: auto; border-radius: 8px; border: 1px solid #e5e4e7; }
  .registered-table { width: 100%; border-collapse: collapse; text-align: left; font-size: 0.95rem; background: #fff; }
  .registered-table th { background: #f8f9fa; color: #34495e; padding: 0.8rem 1rem; border-bottom: 2px solid #e5e4e7; font-weight: bold; }
  .registered-table td { padding: 0.8rem 1rem; border-bottom: 1px solid #eee; vertical-align: middle; }
  .registered-table tr:last-child td { border-bottom: none; }
  
  .tier-cell { font-weight: bold; text-align: center; width: 140px; color: white; }
  .advanced-tier { background-color: #8e44ad; }
  .intermediate-tier { background-color: #2980b9; }
  .beginner-tier { background-color: #27ae60; }

  .player-pool-table { display: flex; flex-wrap: wrap; gap: 0.5rem; align-items: center; }
  .empty-row-text { color: #95a5a6; font-style: italic; font-size: 0.9rem; }

  .player-pool { margin-top: 1rem; padding-top: 0.5rem; display: flex; flex-wrap: wrap; align-items: center; gap: 0.5rem; font-size: 0.9rem; }
  .player-pool.available-pool { margin-top: 0.5rem; padding-top: 0; border-top: none; }
  
  .player-chip { 
    background: #f1f2f6; color: #2f3542; padding: 0.4rem 0.9rem; 
    border-radius: 20px; border: 1px solid #dfe4ea; cursor: pointer; 
    transition: background 0.2s, border-color 0.2s, color 0.2s; font-size: 0.9rem;
    display: inline-flex; align-items: center; gap: 0.4rem;
  }
  .player-chip:hover { background: #e1e5ea; border-color: #3498db; }
  
  .selectable-chip { background: #f8f9fa; border: 1px solid #cbd5e1; }
  
  /* Team 1 Highlight (Blue) */
  .selectable-chip.selected-team1 { background: #3498db; color: white; border-color: #2980b9; font-weight: 500; }
  .selectable-chip.selected-team1 .chip-time { color: #ecf0f1; border-left-color: rgba(255,255,255,0.3); }

  /* Team 2 Highlight (Red) */
  .selectable-chip.selected-team2 { background: #e74c3c; color: white; border-color: #c0392b; font-weight: 500; }
  .selectable-chip.selected-team2 .chip-time { color: #fce4e4; border-left-color: rgba(255,255,255,0.3); }
  
  .checkbox-indicator { font-weight: bold; font-size: 0.8rem; width: 16px; text-align: center; }
  .chip-time { font-size: 0.75rem; color: #7f8c8d; border-left: 1px solid #dcdde1; padding-left: 0.4rem; }

  .helper-hint { font-size: 0.85rem; color: #7f8c8d; margin: 0 0 1rem 0; font-style: italic; }
  .selection-status-group { display: flex; gap: 0.5rem; }
  .team-tag { font-size: 0.8rem; padding: 0.2rem 0.6rem; border-radius: 12px; font-weight: bold; }
  .tag-t1 { background: #ebf5fb; color: #2980b9; border: 1px solid #a9cce3; }
  .tag-t2 { background: #f5b7b1; color: #78281f; border: 1px solid #f1948a; }

  .matchup-container { display: flex; align-items: center; justify-content: space-between; gap: 1rem; background: #f8f9fa; padding: 1.5rem; border-radius: 8px; border: 1px solid #e5e4e7; margin-top: 1rem; }
  .team-box { display: flex; flex-direction: column; gap: 0.8rem; flex: 1; }
  .vs-badge { background: #e74c3c; color: white; font-weight: bold; padding: 0.5rem; border-radius: 50%; width: 30px; height: 30px; display: flex; align-items: center; justify-content: center; font-size: 0.9rem; }

  /* Court Styles */
  .header-with-action { display: flex; justify-content: space-between; align-items: center; }
  .courts-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 1rem; margin-top: 1rem; }
  .court { border: 2px solid #e5e4e7; border-radius: 8px; padding: 1.5rem; display: flex; flex-direction: column; justify-content: space-between; min-height: 140px; transition: all 0.2s ease; }
  .court.available { background: #f8fff9; border-color: #2ecc71; }
  .court.occupied { background: #fff8f8; border-color: #e74c3c; }
  
  .court-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid rgba(0,0,0,0.1); padding-bottom: 0.5rem; margin-bottom: 1rem; }
  .court-name-input { 
    font-size: 1.2rem; font-weight: bold; border: none; background: transparent; 
    color: #2c3e50; padding: 0; margin: 0; width: 60%; outline: none; 
    border-bottom: 1px dashed transparent; transition: border-color 0.2s; 
  }
  .court-name-input:focus, .court-name-input:hover { border-bottom: 1px dashed #3498db; }
  
  .court-header-actions { display: flex; align-items: center; gap: 0.8rem; }
  .status { font-size: 0.85rem; font-weight: bold; text-transform: uppercase; }
  .available .status { color: #27ae60; }
  .occupied .status { color: #c0392b; }
  .delete-icon-btn { background: none; border: none; color: #95a5a6; font-size: 1.2rem; cursor: pointer; padding: 0; line-height: 1; transition: color 0.2s; }
  .delete-icon-btn:hover { color: #e74c3c; }
  
  .empty-text { color: #7f8c8d; font-style: italic; text-align: center; margin: 1rem 0; }
  .playing { text-align: center; font-weight: bold; font-size: 1.1rem; color: #2c3e50; margin: 0 0 0.5rem 0; }
  .time-started { text-align: center; font-size: 0.85rem; color: #e67e22; font-weight: bold; margin: 0 0 1rem 0; }
  .clear-btn { width: 100%; background: #e74c3c; color: white; border: none; padding: 0.8rem; border-radius: 6px; cursor: pointer; font-weight: bold; }
  
  /* Queue Styles */
  .queue-list { display: flex; flex-direction: column; gap: 0.8rem; margin-top: 1rem; }
  .queue-item { display: flex; justify-content: space-between; align-items: center; background: #f8f9fa; padding: 1rem; border-radius: 8px; border-left: 4px solid #3498db; }
  .team-info { display: flex; align-items: center; }
  .queue-number { font-weight: bold; color: #7f8c8d; margin-right: 1.5rem; font-size: 1.2rem; }
  .team-name { font-size: 1.1rem; font-weight: 600; color: #2c3e50; margin-bottom: 0.3rem; }
  .match-badge { background: #e8f4f8; color: #2980b9; padding: 0.2rem 0.6rem; border-radius: 20px; font-size: 0.8rem; display: inline-block; }
  
  .queue-actions { display: flex; gap: 0.5rem; }
  .edit-btn { background: #f39c12; color: white; border: none; padding: 0.8rem 1rem; border-radius: 6px; font-weight: bold; cursor: pointer; transition: background 0.2s; }
  .edit-btn:hover { background: #e67e22; }
  .play-btn { background: #2ecc71; color: white; border: none; padding: 0.8rem 1.2rem; border-radius: 6px; font-weight: bold; cursor: pointer; transition: transform 0.1s; }
  .play-btn:hover { background: #27ae60; transform: scale(1.02); }

  /* History Styles */
  .history-list { display: flex; flex-direction: column; gap: 0.8rem; }
  .history-item { display: flex; justify-content: space-between; align-items: center; background: #fafafa; padding: 1rem; border-radius: 8px; border: 1px solid #eee; }
  .history-teams { font-weight: 600; color: #2c3e50; font-size: 1.05rem; }
  .history-times { color: #95a5a6; font-size: 0.85rem; margin-top: 0.3rem; }
  .history-score-badge { background: #2c3e50; color: white; padding: 0.5rem 1rem; border-radius: 8px; font-weight: bold; font-size: 1.2rem; }

  /* Modal Styles */
  .modal-backdrop { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.5); z-index: 1000; }
  .modal { position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); background: white; padding: 2rem; border-radius: 12px; box-shadow: 0 10px 25px rgba(0,0,0,0.2); z-index: 1001; width: 90%; max-width: 600px; }
  .modal-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #eee; padding-bottom: 1rem; margin-bottom: 1.5rem; }
  .modal-header h2 { margin: 0; }
  .close-icon-btn { background: none; border: none; font-size: 1.5rem; color: #7f8c8d; cursor: pointer; }
  .close-icon-btn:hover { color: #e74c3c; }
  .modal-matchup { margin: 0 0 2rem 0; }
  .modal-footer { display: flex; justify-content: space-between; align-items: center; border-top: 1px solid #eee; padding-top: 1.5rem; }
  .modal-right-actions { display: flex; gap: 1rem; }

  /* Score Modal specific */
  .score-box { display: flex; flex-direction: column; align-items: center; gap: 0.5rem; flex: 1; }
  .score-box label { font-weight: bold; color: #34495e; }
  .score-input { font-size: 1.5rem; text-align: center; padding: 1rem; font-weight: bold; border: 2px solid #3498db; }

  @media (max-width: 600px) {
    .matchup-container { flex-direction: column; }
    .vs-badge { margin: 0.5rem 0; }
    .header-with-action { flex-direction: column; align-items: flex-start; gap: 1rem; }
    .queue-item { flex-direction: column; align-items: flex-start; gap: 1rem; }
    .queue-actions { width: 100%; display: flex; justify-content: stretch; }
    .queue-actions button { flex: 1; }
    .modal-footer { flex-direction: column-reverse; gap: 1rem; }
    .modal-right-actions { width: 100%; justify-content: space-between; }
    .danger-btn { width: 100%; }
  }
</style>