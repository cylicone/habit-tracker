<script lang="ts">
  import { onMount } from "svelte";
  import "../app.css";
  import Database from "@tauri-apps/plugin-sql";

  // Interface for a habit, where `completed` is derived from today's master record
  interface Habit {
    id: number;
    name: string;
    streak: number;
    completed: boolean;
  }

  let testDateOverride = "2025-4-14"; // Leave empty in production

  function getEffectiveDate(): string {
    return testDateOverride || new Date().toISOString().split("T")[0];
  }

  let habits: Habit[] = [];
  let newHabitName = "";
  let isAddingHabit = false;
  let db: any;

  // Initialize the database and create tables if they don't already exist.
  async function initDatabase() {
    db = await Database.load("sqlite:mydata.db");

    // The habits table no longer stores the completed value.
    await db.execute(`
      CREATE TABLE IF NOT EXISTS habits (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        streak INTEGER NOT NULL DEFAULT 0
      )
    `);

    // The master table logs daily habit activity.
    await db.execute(`
      CREATE TABLE IF NOT EXISTS master (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        habit_id INTEGER NOT NULL,
        completed INTEGER NOT NULL DEFAULT 0,
        completed_repeats INTEGER NOT NULL DEFAULT 0,
        date STRING
      )
    `);

    return db;
  }

  // Fetch habits and merge them with today's master records.
  async function getHabits(db: any): Promise<Habit[]> {
    const habitRows = await db.select("SELECT * FROM habits");
    const today = new Date().toISOString().split("T")[0]; // e.g., "2025-04-14"
    // Grab today's master records (if any) for each habit.
    const masterRows = await db.select(
      "SELECT * FROM master WHERE date LIKE $1",
      [`${today}%`],
    );

    const mappedHabits: Habit[] = habitRows.map((row: any) => {
      // Find today's master record for this habit, if any.
      const masterRecord = masterRows.find((m: any) => m.habit_id == row.id);
      return {
        id: row.id,
        name: row.name,
        streak: row.streak,
        completed: masterRecord ? Boolean(masterRecord.completed) : false,
      };
    });

    return mappedHabits;
  }

  onMount(async () => {
    try {
      db = await initDatabase();
      habits = await getHabits(db);
      await printTables(); // For debugging, print the current tables.
    } catch (err) {
      console.error("Database initialization error:", err);
    }
  });

  // Reactive statements to calculate progress.
  $: completedCount = habits.filter((habit) => habit.completed).length;
  $: progressPercent = habits.length
    ? (completedCount / habits.length) * 100
    : 0;

  // Add new reactive variables for UI state
  $: progressColor = progressPercent >= 80 ? 'bg-green-500' : 
                     progressPercent >= 50 ? 'bg-blue-500' : 
                     progressPercent >= 30 ? 'bg-yellow-500' : 'bg-red-500';

  // Debug function: print both tables to the console.
  async function printTables() {
    try {
      const habitsRows = await db.select("SELECT * FROM habits");
      console.log("Habits Table:", habitsRows);
      const masterRows = await db.select("SELECT * FROM master");
      console.log("Master Table:", masterRows);
    } catch (err) {
      console.error("Error printing tables:", err);
    }
  }

  // Toggle a habit: update (or insert) the master record for today and adjust the streak.
  async function toggleHabit(id: number) {
    const habitIndex = habits.findIndex((h) => h.id == id);
    if (habitIndex === -1) return;

    const currentDate = new Date().toISOString(); // full timestamp
    const today = currentDate.split("T")[0];

    // Look for an existing master record for this habit for today.
    const masterRecords = await db.select(
      "SELECT * FROM master WHERE habit_id = $1 AND date LIKE $2 LIMIT 1",
      [id, `${today}%`],
    );

    let updatedCompleted: boolean;
    if (masterRecords.length > 0) {
      // Found an existing entry – toggle its completed status.
      const currentStatus = Boolean(masterRecords[0].completed);
      updatedCompleted = !currentStatus;
      await db.execute(
        "UPDATE master SET completed = $1, date = $2 WHERE id = $3",
        [updatedCompleted ? 1 : 0, currentDate, masterRecords[0].id],
      );
    } else {
      // No entry exists: create one with completed status set to true.
      updatedCompleted = true;
      await db.execute(
        "INSERT INTO master (habit_id, completed, date) VALUES ($1, $2, $3)",
        [id, 1, currentDate],
      );
    }

    // Update the streak in the habits table.
    // Increase if toggled to completed, decrease (to a minimum of 0) if toggled off.
    let newStreak = habits[habitIndex].streak;
    if (updatedCompleted) {
      newStreak = newStreak + 1;
    } else {
      newStreak = Math.max(newStreak - 1, 0);
    }
    await db.execute("UPDATE habits SET streak = $1 WHERE id = $2", [
      newStreak,
      id,
    ]);

    // Update the local state.
    habits[habitIndex] = {
      ...habits[habitIndex],
      completed: updatedCompleted,
      streak: newStreak,
    };

    await printTables(); // Debug: print tables after update.
  }

  // Add a new habit to the habits table.
  async function addHabit() {
    if (!newHabitName.trim()) return;
    try {
      await db.execute("INSERT INTO habits (name, streak) VALUES ($1, $2)", [
        newHabitName.trim(),
        0,
      ]);
      habits = await getHabits(db);
      newHabitName = "";
      isAddingHabit = false;
      await printTables(); // Debug: print tables after adding.
    } catch (err) {
      console.error("Error adding habit:", err);
    }
  }

  // Delete a habit from the database.
  async function deleteHabit(id: number) {
    if (confirm("Are you sure you want to delete this habit?")) {
      try {
        await db.execute("DELETE FROM habits WHERE id = $1", [id]);
        // Optionally, also remove related master entries.
        await db.execute("DELETE FROM master WHERE habit_id = $1", [id]);
        habits = habits.filter((habit) => habit.id !== id);
        await printTables(); // Debug: print tables after deletion.
      } catch (err) {
        console.error("Error deleting habit:", err);
      }
    }
  }
</script>

<main class="min-h-screen bg-gradient-to-b from-[#0A0A0A] to-[#121218] text-white flex flex-col items-center px-4 py-8 max-w-3xl mx-auto">
  <!-- Enhanced Header with Date and Stats Card -->
  <header class="w-full mb-8">
    <div class="flex items-center justify-between mb-6">
      <h1 class="text-3xl font-bold bg-gradient-to-r from-blue-400 via-purple-400 to-pink-400 bg-clip-text text-transparent">
        Daily Habits
      </h1>
      <a
        href="/previous-days"
        class="px-4 py-2 bg-[#1A1A1A] text-white rounded-lg 
               hover:bg-[#252525] transition-all duration-300 flex items-center gap-2 
               border border-indigo-500/20 hover:border-indigo-500/40
               hover:shadow-[0_0_15px_rgba(99,102,241,0.1)]"
      >
        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-indigo-400" viewBox="0 0 20 20" fill="currentColor">
          <path fill-rule="evenodd" d="M6 2a1 1 0 00-1 1v1H4a2 2 0 00-2 2v10a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-1V3a1 1 0 10-2 0v1H7V3a1 1 0 00-1-1zm0 5a1 1 0 000 2h8a1 1 0 100-2H6z" clip-rule="evenodd" />
        </svg>
        <span class="bg-gradient-to-r from-indigo-400 to-purple-400 bg-clip-text text-transparent font-medium">
          History
        </span>
      </a>
    </div>

    <!-- Stats Card -->
    <div class="bg-[#1A1A1A] rounded-xl p-6 border border-purple-500/20 
                hover:border-purple-500/30 transition-all duration-500
                hover:shadow-[0_0_20px_rgba(168,85,247,0.1)]">
      <div class="flex items-center justify-between mb-4">
        <div>
          <h2 class="text-xl font-semibold text-white mb-1">Today's Progress</h2>
          <p class="text-[#888888]">
            {new Date().toLocaleDateString("en-US", {
              weekday: "long",
              month: "long",
              day: "numeric"
            })}
          </p>
        </div>
        <div class="text-2xl font-bold text-white">
          {Math.round(progressPercent)}%
        </div>
      </div>
      
      <!-- Enhanced Progress Bar -->
      <div class="w-full h-4 bg-[#252525] rounded-full overflow-hidden">
        <div
          class="h-full transition-all duration-500 ease-out {progressColor}"
          style="width: {progressPercent}%; transition: width 1s ease-in-out;"
        ></div>
      </div>
      
      <div class="mt-4 text-[#888888]">
        <span class="font-medium text-white">{completedCount}</span> of <span class="font-medium text-white">{habits.length}</span> habits completed
      </div>
    </div>
  </header>

  <!-- Habits List with Enhanced Styling -->
  <section class="w-full space-y-3 mb-8">
    {#each habits as habit (habit.id)}
      <div
        class="group bg-[#1A1A1A] px-6 py-4 rounded-xl
               hover:bg-[#252525] active:bg-[#2A2A2A] 
               transition-all duration-300 flex items-center 
               justify-between border border-blue-500/20
               hover:border-blue-500/30 cursor-pointer
               hover:shadow-[0_0_15px_rgba(59,130,246,0.1)]"
        on:click={() => toggleHabit(habit.id)}
      >
        <div class="flex items-center space-x-4 flex-1">
          <button
            class="relative flex items-center justify-center h-7 w-7
                   focus:outline-none transform transition-all
                   hover:scale-110 active:scale-95"
            on:click|stopPropagation={() => toggleHabit(habit.id)}
          >
            <div
              class="absolute inset-0 rounded-full border-2 transition-all duration-300
                     {habit.completed ? 'border-emerald-500 bg-emerald-500/10' : 'border-[#444444] group-hover:border-[#555555]'}"
            ></div>
            {#if habit.completed}
              <!-- Checkmark -->
              <svg 
                class="w-4 h-4 text-emerald-500 transform transition-all duration-300 animate-check" 
                fill="none" 
                stroke="currentColor" 
                viewBox="0 0 24 24"
              >
                <path 
                  stroke-linecap="round" 
                  stroke-linejoin="round" 
                  stroke-width="3" 
                  d="M5 13l4 4L19 7"
                ></path>
              </svg>
              <!-- Green fill effect -->
              <div 
                class="absolute inset-0 rounded-full bg-gradient-to-br from-emerald-500/20 to-green-500/20 
                       animate-fill-in"
              ></div>
            {/if}
          </button>
          
          <span class="text-lg transition-all duration-300 {habit.completed ? 
            'text-[#666666] line-through' : 'text-white group-hover:text-blue-100'}">
            {habit.name}
          </span>
        </div>
        
        <div class="flex items-center space-x-4">
          <div class="px-3 py-1 rounded-full bg-[#252525] 
                      border border-cyan-500/20 hover:border-cyan-500/40
                      hover:bg-[#2A2A2A] active:bg-[#303030] 
                      transition-all duration-300
                      hover:shadow-[0_0_10px_rgba(34,211,238,0.1)]">
            <span class="slot-number text-cyan-400">{habit.streak}</span>
            <span class="text-xs ml-1 text-[#888888]">days</span>
          </div>
          
          <button
            class="text-[#666666] hover:text-rose-400 active:text-rose-500 
                   transition-all duration-300 p-2 rounded-lg
                   hover:bg-rose-500/10 active:bg-rose-500/20"
            on:click|stopPropagation={() => deleteHabit(habit.id)}
            title="Delete Habit"
          >
            <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd"/>
            </svg>
          </button>
        </div>
      </div>
    {/each}
  </section>

  <!-- Enhanced Add Habit Interface -->
  {#if isAddingHabit}
    <div class="w-full bg-[#1A1A1A] rounded-xl p-6 
                border border-purple-500/20 hover:border-purple-500/30
                transition-all duration-300
                hover:shadow-[0_0_20px_rgba(168,85,247,0.1)]">
      <input
        type="text"
        bind:value={newHabitName}
        placeholder="What habit would you like to track?"
        class="w-full bg-[#252525] text-white px-4 py-3 rounded-lg mb-4 
               focus:outline-none focus:ring-2 focus:ring-purple-500/50 
               focus:border-purple-500/50 transition-all duration-300
               placeholder-[#666666] text-lg border border-[#444444]
               hover:border-purple-500/30"
        on:keydown={(e) => e.key === "Enter" && addHabit()}
      />
      <div class="flex space-x-3">
        <button
          class="flex-1 py-3 rounded-lg bg-gradient-to-r from-blue-600 to-purple-600 
                 text-white font-medium hover:from-blue-500 hover:to-purple-500 
                 active:from-blue-700 active:to-purple-700
                 transition-all duration-300 transform
                 hover:translate-y-[-1px] active:translate-y-[1px]
                 hover:shadow-[0_0_15px_rgba(147,51,234,0.3)]
                 focus:outline-none focus:ring-2 focus:ring-purple-500/50 
                 focus:ring-offset-2 focus:ring-offset-[#1A1A1A]"
          on:click={addHabit}
        >
          Add Habit
        </button>
        <button
          class="flex-1 py-3 rounded-lg bg-[#252525] text-[#888888] font-medium 
                 hover:bg-[#333333] active:bg-[#383838] 
                 transition-all duration-300 border border-[#444444]
                 transform hover:translate-y-[-1px] active:translate-y-[1px]
                 focus:outline-none focus:ring-2 focus:ring-[#444444]"
          on:click={() => {
            isAddingHabit = false;
            newHabitName = "";
          }}
        >
          Cancel
        </button>
      </div>
    </div>
  {:else}
    <button
      class="w-full py-4 rounded-xl bg-[#1A1A1A] text-[#888888] font-medium 
             hover:bg-[#252525] active:bg-[#2A2A2A] 
             transition-all duration-300 border border-blue-500/20
             hover:border-blue-500/30 text-lg
             transform hover:translate-y-[-1px] active:translate-y-[1px]
             hover:shadow-[0_0_15px_rgba(59,130,246,0.1)]
             focus:outline-none focus:ring-2 focus:ring-blue-500/50"
      on:click={() => (isAddingHabit = true)}
    >
      + Create New Habit
    </button>
  {/if}
</main>

<style>
  .slot-number {
    display: inline-block;
    min-width: 1.5rem;
    text-align: center;
    font-weight: bold;
    animation: slotSpin 0.8s ease-out;
  }
  
  @keyframes slotSpin {
    0% {
      transform: translateY(-100%);
      opacity: 0;
    }
    60% {
      transform: translateY(10%);
      opacity: 1;
    }
    100% {
      transform: translateY(0);
      opacity: 1;
    }
  }

  /* Add new animation for checkbox */
  @keyframes scale-in {
    from {
      transform: scale(0);
    }
    to {
      transform: scale(1);
    }
  }

  .animate-scale-in {
    animation: scale-in 0.2s ease-out forwards;
  }

  @keyframes check {
    0% {
      transform: scale(0) rotate(45deg);
      opacity: 0;
    }
    50% {
      transform: scale(1.2) rotate(-5deg);
      opacity: 0.8;
    }
    100% {
      transform: scale(1) rotate(0);
      opacity: 1;
    }
  }

  @keyframes fill-in {
    0% {
      opacity: 0;
      transform: scale(0.5);
    }
    50% {
      opacity: 0.4;
      transform: scale(1.1);
    }
    100% {
      opacity: 1;
      transform: scale(1);
    }
  }

  .animate-check {
    animation: check 0.3s ease-out forwards;
  }

  .animate-fill-in {
    animation: fill-in 0.3s ease-out forwards;
  }
</style>
