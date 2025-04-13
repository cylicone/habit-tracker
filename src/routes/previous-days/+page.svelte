<script lang="ts">
  import { onMount } from "svelte";
  import Database from "@tauri-apps/plugin-sql";

  interface Habit {
    id: number;
    name: string;
    streak: number;
    completed: boolean;
  }

  let selectedDate = new Date().toISOString().split("T")[0];
  let habitsForDate: Habit[] = [];
  let db: any;

  async function initDatabase() {
    db = await Database.load("sqlite:mydata.db");

    await db.execute(`
      CREATE TABLE IF NOT EXISTS habits (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        streak INTEGER NOT NULL DEFAULT 0
      )
    `);

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

  // get habits and merge them with the master records for the given date.
  async function getHabitsForDate(date: string): Promise<Habit[]> {
    const habitRows = await db.select("SELECT * FROM habits");
    const masterRows = await db.select(
      "SELECT * FROM master WHERE date LIKE $1",
      [`${date}%`],
    );

    const mappedHabits: Habit[] = habitRows.map((row: any) => {
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

  async function fetchHabits() {
    if (!db) return;
    habitsForDate = await getHabitsForDate(selectedDate);
  }

  onMount(async () => {
    try {
      db = await initDatabase();
      await fetchHabits();
    } catch (err) {
      console.error("Database initialization error:", err);
    }
  });

  function handleDateChange(e: Event) {
    const input = e.target as HTMLInputElement;
    selectedDate = input.value;
    fetchHabits();
  }

  // Calculate progress from the habits of the selected date.
  $: completedCount = habitsForDate.filter((habit) => habit.completed).length;
  $: progressPercent = habitsForDate.length
    ? (completedCount / habitsForDate.length) * 100
    : 0;
  $: progressColor =
    progressPercent >= 80
      ? "bg-green-500"
      : progressPercent >= 50
        ? "bg-blue-500"
        : progressPercent >= 30
          ? "bg-yellow-500"
          : "bg-red-500";

  async function toggleHabit(id: number) {
    const habitIndex = habitsForDate.findIndex((h) => h.id == id);
    if (habitIndex === -1) return;

    const currentDate = new Date(selectedDate).toISOString();
    const datePrefix = selectedDate; // YYYY-MM-DD

    // Look for an existing master record for this habit for the selected date
    const masterRecords = await db.select(
      "SELECT * FROM master WHERE habit_id = $1 AND date LIKE $2 LIMIT 1",
      [id, `${datePrefix}%`],
    );

    let updatedCompleted: boolean;
    if (masterRecords.length > 0) {
      const currentStatus = Boolean(masterRecords[0].completed);
      updatedCompleted = !currentStatus;
      await db.execute(
        "UPDATE master SET completed = $1, date = $2 WHERE id = $3",
        [updatedCompleted ? 1 : 0, currentDate, masterRecords[0].id],
      );
    } else {
      updatedCompleted = true;
      await db.execute(
        "INSERT INTO master (habit_id, completed, date) VALUES ($1, $2, $3)",
        [id, 1, currentDate],
      );
    }

    const habit = habitsForDate[habitIndex];
    let newStreak = habit.streak;
    if (updatedCompleted) {
      newStreak = newStreak + 1;
    } else {
      newStreak = Math.max(newStreak - 1, 0);
    }
    await db.execute("UPDATE habits SET streak = $1 WHERE id = $2", [
      newStreak,
      id,
    ]);

    habitsForDate[habitIndex] = {
      ...habitsForDate[habitIndex],
      completed: updatedCompleted,
      streak: newStreak,
    };
    habitsForDate = [...habitsForDate];
  }
</script>

<main
  class="min-h-screen bg-gradient-to-b from-[#0A0A0A] to-[#121218] text-white
         flex flex-col items-center px-4 py-8 max-w-3xl mx-auto"
>
  <header class="w-full mb-8">
    <div class="flex items-center justify-between mb-6">
      <button
        on:click={() => window.history.back()}
        class="px-4 py-2 bg-[#1A1A1A] text-white rounded-lg
               hover:bg-[#252525] transition-all duration-300 flex items-center gap-2
               border border-indigo-500/20 hover:border-indigo-500/40
               hover:shadow-[0_0_15px_rgba(99,102,241,0.1)]"
        title="Go Back"
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          class="h-5 w-5 text-indigo-400"
          viewBox="0 0 20 20"
          fill="currentColor"
        >
          <path
            fill-rule="evenodd"
            d="M9.707 16.707a1 1 0 01-1.414 0l-6-6a1 1 0 010-1.414l6-6a1 1 0 011.414 1.414L5.414 9H17a1 1 0 110 2H5.414l4.293 4.293a1 1 0 010 1.414z"
            clip-rule="evenodd"
          />
        </svg>
        <span
          class="bg-gradient-to-r from-indigo-400 to-purple-400 bg-clip-text text-transparent font-medium"
        >
          Back
        </span>
      </button>
      <h1
        class="text-3xl font-bold bg-gradient-to-r from-blue-400 via-purple-400 to-pink-400 bg-clip-text text-transparent"
      >
        History
      </h1>
      <div class="w-[88px]"></div>
    </div>
  </header>

  <section class="w-full mb-8">
    <div
      class="bg-[#1A1A1A] rounded-xl p-6 border border-purple-500/20
                hover:border-purple-500/30 transition-all duration-500
                hover:shadow-[0_0_20px_rgba(168,85,247,0.1)]"
    >
      <div class="flex items-center justify-between mb-6">
        <label for="date" class="text-lg font-medium text-white"
          >Select Date:</label
        >
        <input
          type="date"
          id="date"
          bind:value={selectedDate}
          on:change={handleDateChange}
          class="bg-[#252525] text-white px-4 py-2 rounded-lg
                 focus:outline-none focus:ring-2 focus:ring-purple-500/50
                 border border-[#444444] hover:border-purple-500/30
                 transition-all duration-300"
        />
      </div>

      <div class="flex items-center justify-between mb-4">
        <div>
          <h2 class="text-xl font-semibold text-white mb-1">Day's Progress</h2>
          <p class="text-[#888888]">
            {new Date(selectedDate).toLocaleDateString("en-US", {
              weekday: "long",
              month: "long",
              day: "numeric",
            })}
          </p>
        </div>
        <div class="text-2xl font-bold text-white">
          {Math.round(progressPercent)}%
        </div>
      </div>

      <div class="w-full h-4 bg-[#252525] rounded-full overflow-hidden">
        <div
          class="h-full transition-all duration-500 ease-out {progressColor}"
          style="width: {progressPercent}%; transition: width 1s ease-in-out;"
        ></div>
      </div>

      <div class="mt-4 text-[#888888]">
        <span class="font-medium text-white">{completedCount}</span> of
        <span class="font-medium text-white">{habitsForDate.length}</span> habits
        completed
      </div>
    </div>
  </section>

  <section class="w-full space-y-3">
    {#each habitsForDate as habit (habit.id)}
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
                     {habit.completed
                ? 'border-emerald-500 bg-emerald-500/10'
                : 'border-[#444444] group-hover:border-[#555555]'}"
            ></div>
            {#if habit.completed}
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
              <div
                class="absolute inset-0 rounded-full bg-gradient-to-br from-emerald-500/20 to-green-500/20
                       animate-fill-in"
              ></div>
            {/if}
          </button>

          <span
            class="text-lg transition-all duration-300 {habit.completed
              ? 'text-[#666666] line-through'
              : 'text-white group-hover:text-blue-100'}"
          >
            {habit.name}
          </span>
        </div>

        <div class="flex items-center space-x-4">
          <div
            class="px-3 py-1 rounded-full bg-[#252525]
                      border border-cyan-500/20 hover:border-cyan-500/40
                      hover:bg-[#2A2A2A] active:bg-[#303030]
                      transition-all duration-300
                      hover:shadow-[0_0_10px_rgba(34,211,238,0.1)]"
          >
            <span class="slot-number text-cyan-400">{habit.streak}</span>
            <span class="text-xs ml-1 text-[#888888]">days</span>
          </div>
        </div>
      </div>
    {/each}

    {#if habitsForDate.length === 0}
      <div class="text-center py-8 text-gray-400">
        No habits found for this date
      </div>
    {/if}
  </section>
</main>

<style>
  input[type="date"]::-webkit-calendar-picker-indicator {
    filter: invert(1);
    opacity: 0.5;
    cursor: pointer;
  }

  input[type="date"]::-webkit-calendar-picker-indicator:hover {
    opacity: 0.8;
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
