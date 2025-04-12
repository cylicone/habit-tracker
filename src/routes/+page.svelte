<script lang="ts">
  import { onMount } from "svelte";
  import "../app.css";
  import Database from "@tauri-apps/plugin-sql";

  interface Habit {
    id: number;
    name: string;
    completed: boolean;
    createdAt: string;
    streak: number;
  }

  let habits: Habit[] = [];
  let newHabitName = "";
  let isAddingHabit = false;
  let db: any;

  async function initDatabase() {
    const db = await Database.load("sqlite:mydata.db");
    await db.execute(`
      CREATE TABLE IF NOT EXISTS habits (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        completed INTEGER NOT NULL DEFAULT 0,
        last_updated TEXT,
        streak INTEGER
        last_streak_increment TEXT
      )
    `);
    return db;
  }

  async function getHabits(db: any): Promise<Habit[]> {
    const rows = await db.select("SELECT * FROM habits");
    const mappedHabits = rows.map((row: any) => ({
      id: row.id,
      name: row.name,
      completed: Boolean(row.completed),
      createdAt: row.last_updated,
      streak: row.streak,
    }));
    return mappedHabits;
  }

  onMount(async () => {
    try {
      db = await initDatabase();
      habits = await getHabits(db);
    } catch (err) {
      console.error("Database initialization error:", err);
    }
  });

  $: completedCount = habits.filter((habit) => habit.completed).length;
  $: progressPercent = habits.length
    ? (completedCount / habits.length) * 100
    : 0;

  async function toggleHabit(id: number) {
    habits = habits.map((habit) => {
      if (habit.id === id) {
        const updatedHabit = { ...habit, completed: !habit.completed };
        db.execute(
          "UPDATE habits SET completed = $1, last_updated = $2, streak = $3 WHERE id = $4",
          [
            updatedHabit.completed ? 1 : 0,
            new Date().toISOString(),
            habit.streak + 1,
            id,
          ],
        );
        return updatedHabit;
      }
      return habit;
    });
  }

  async function addHabit() {
    if (!newHabitName.trim()) return;
    const now = new Date().toISOString();
    await db.execute(
      "INSERT INTO habits (name, completed, last_updated) VALUES ($1, $2, $3)",
      [newHabitName.trim(), 0, now],
    );
    habits = await getHabits(db);
    newHabitName = "";
    isAddingHabit = false;
  }

  async function deleteHabit(id: number) {
    if (confirm("Are you sure you want to delete this habit?")) {
      await db.execute("DELETE FROM habits WHERE id = $1", [id]);
      habits = habits.filter((habit) => habit.id !== id);
    }
  }
</script>

<main
  class="min-h-screen bg-[#121212] text-white flex flex-col items-center px-4 py-6 max-w-2xl mx-auto"
>
  <header class="w-full flex items-center justify-between mb-6">
    <h1 class="text-2xl font-bold">Habits</h1>
    <div class="text-sm text-gray-400">
      {new Date().toLocaleDateString("en-US", {
        weekday: "long",
        month: "long",
        day: "numeric",
      })}
    </div>
  </header>

  <section class="w-full mb-6">
    <p class="text-gray-400 text-sm mb-2">Today's Progress</p>
    <div class="flex items-center justify-between mb-2">
      <span class="text-lg font-medium"
        >{completedCount}/{habits.length} completed</span
      >
      <span class="text-gray-400">{Math.round(progressPercent)}%</span>
    </div>
    <div class="w-full h-3 bg-[#1C1C1E] rounded-full overflow-hidden">
      <div
        class="h-full bg-gradient-to-r from-blue-500 to-purple-500 transition-all duration-300"
        style="width: {progressPercent}%"
      ></div>
    </div>
  </section>

  <section class="w-full space-y-4 mb-6">
    {#each habits as habit, i (habit.id)}
      <div
        class="group bg-[#1C1C1E] px-4 py-3 rounded-lg hover:bg-[#242426] transition-all flex items-center justify-between"
      >
        <div class="flex items-center space-x-3 flex-1">
          <button
            class="relative flex items-center justify-center h-6 w-6 cursor-pointer"
            on:click={() => toggleHabit(habit.id)}
          >
            <div
              class="absolute inset-0 rounded-full border-2 border-gray-500 {habit.completed
                ? 'border-blue-500'
                : ''}"
            ></div>
            {#if habit.completed}
              <div class="h-4 w-4 bg-blue-500 rounded-full"></div>
            {/if}
          </button>
          <span
            class="text-base {habit.completed
              ? 'text-gray-500 line-through'
              : ''}">{habit.name}</span
          >
        </div>
        <div class="flex items-center space-x-2">
          <span class="slot-number">{habit.streak}</span>
          <button
            class="ml-4 opacity-100 ransition-opacity text-red-500 hover:text-red-400"
            on:click={() => deleteHabit(habit.id)}
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              class="h-5 w-5"
              viewBox="0 0 20 20"
              fill="currentColor"
            >
              <path
                fill-rule="evenodd"
                d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z"
                clip-rule="evenodd"
              />
            </svg>
          </button>
        </div>
      </div>
    {/each}
  </section>

  {#if isAddingHabit}
    <div class="w-full bg-[#1C1C1E] rounded-lg p-4">
      <input
        type="text"
        bind:value={newHabitName}
        placeholder="Enter habit name..."
        class="w-full bg-[#2D2D2F] text-white px-4 py-2 rounded-lg mb-3 focus:outline-none focus:ring-2 focus:ring-blue-500"
        on:keydown={(e) => e.key === "Enter" && addHabit()}
      />
      <div class="flex space-x-2">
        <button
          class="flex-1 py-2 rounded-lg bg-blue-500 text-white font-medium hover:bg-blue-600 transition-colors"
          on:click={addHabit}>Add</button
        >
        <button
          class="flex-1 py-2 rounded-lg bg-[#2D2D2F] text-gray-300 font-medium hover:bg-[#3a3a3c] transition-colors"
          on:click={() => {
            isAddingHabit = false;
            newHabitName = "";
          }}>Cancel</button
        >
      </div>
    </div>
  {:else}
    <button
      class="w-full py-3 rounded-lg bg-[#2D2D2F] text-gray-200 font-medium hover:bg-[#3a3a3c] transition-colors"
      on:click={() => (isAddingHabit = true)}>+ Add habit</button
    >
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
</style>
