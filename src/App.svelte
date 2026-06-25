<script lang="ts">
    import { fly } from "svelte/transition";
    import Bracket, {
        type AgeGroup,
        type Sex,
        type Sport,
    } from "./lib/Bracket.svelte";
    import { app } from "./lib/firebase";

    interface TournamentOption {
        id: string;
        label: string;
        ageGroup: AgeGroup;
        sex: Sex;
        doubleElim?: boolean;
    }

    const tournamentConfig: Record<Sport, TournamentOption[]> = {
        soccer: [
            {
                id: "m_under10",
                label: "U10 Kids",
                ageGroup: "under10",
                sex: "male",
            },
            {
                id: "m_under14",
                label: "U14 Kids",
                ageGroup: "under14",
                sex: "male",
            },
            {
                id: "m_under18",
                label: "U18 Boys",
                ageGroup: "under18",
                sex: "male",
            },
            {
                id: "m_open",
                label: "Mens Local",
                ageGroup: "open",
                sex: "male",
            },
            {
                id: "m_international",
                label: "Mens International",
                ageGroup: "open",
                sex: "male",
                doubleElim: true,
            },
        ],
        volleyball: [
            {
                id: "f_under18",
                label: "U18 Girls",
                ageGroup: "under18",
                sex: "female",
            },
            {
                id: "f_open",
                label: "Womens Open",
                ageGroup: "open",
                sex: "female",
            },
            {
                id: "m_under18",
                label: "U18 Boys",
                ageGroup: "under18",
                sex: "male",
            },
            { id: "m_open", label: "Mens Open", ageGroup: "open", sex: "male" },
        ],
        basketball: [
            { id: "m_open", label: "Mens Open", ageGroup: "open", sex: "male" },
        ],
    };

    const sports = Object.keys(tournamentConfig) as Sport[];

    let currentSport = $state<Sport>("soccer"),
        availableGroups = $derived(tournamentConfig[currentSport]),
        selectedOptionIndex = $state<number>(0),
        currentOption = $derived(
            availableGroups[selectedOptionIndex] || availableGroups[0],
        );

    let useColorLines = $state(false);

    $effect(() => {
        if (selectedOptionIndex >= availableGroups.length) {
            selectedOptionIndex = 0;
        }
    });
</script>

<div class="dashboard-layout">
    <nav class="top-nav">
        <div class="nav-container">
            <div class="nav-section">
                <span class="nav-label">Sport:</span>
                <div class="btn-group">
                    {#each sports as sport}
                        <button
                            class="nav-btn"
                            class:active={currentSport === sport}
                            onclick={() => {
                                currentSport = sport;
                                selectedOptionIndex = 0;
                            }}
                        >
                            {sport.charAt(0).toUpperCase() + sport.slice(1)}
                        </button>
                    {/each}
                </div>
            </div>

            <div class="nav-section">
                <span class="nav-label">Group:</span>
                <div class="btn-group">
                    {#each availableGroups as group, index}
                        <button
                            class="nav-btn"
                            class:active={selectedOptionIndex === index}
                            onclick={() => (selectedOptionIndex = index)}
                        >
                            {group.label}
                        </button>
                    {/each}
                </div>
            </div>

            {#if currentOption?.doubleElim}
                <div class="nav-section control-fade-in">
                    <span class="nav-label">Paths:</span>
                    <label class="nav-toggle-switch">
                        <input type="checkbox" bind:checked={useColorLines} />
                        <span class="toggle-slider"></span>
                        <span class="toggle-caption-text">Chromatic Routes</span
                        >
                    </label>
                </div>
            {/if}
        </div>
    </nav>

    <main class="content-area">
        {#if currentOption}
            {#key currentOption.id}
                <div
                    class="bracket-transition-layer"
                    in:fly={{ x: 160, duration: 250, delay: 150 }}
                    out:fly={{ x: -160, duration: 200 }}
                >
                    <Bracket
                        divisionLabel={currentOption.label}
                        sport={currentSport}
                        ageGroup={currentOption.ageGroup}
                        sex={currentOption.sex}
                        isDoubleElimination={currentOption.doubleElim
                            ? true
                            : false}
                        totalTeams={Math.ceil(Math.random() * 16)}
                        {useColorLines}
                    />
                </div>
            {/key}
        {/if}
    </main>
</div>

<style>
    :global(body) {
        margin: 0;
        background-color: #f7f9fa;
    }

    .dashboard-layout {
        display: flex;
        flex-direction: column;
        min-height: 100vh;
    }

    .top-nav {
        background-color: #ffffff;
        border-bottom: 1px solid #e2e8f0;
        padding: 1rem 2rem;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
    }

    .nav-container {
        max-width: 1200px;
        margin: 0 auto;
        display: flex;
        flex-wrap: wrap;
        gap: 2.5rem;
        align-items: center;
    }

    .nav-section {
        display: flex;
        align-items: center;
        gap: 0.75rem;
    }

    .nav-label {
        font-size: 0.85rem;
        font-weight: 700;
        text-transform: uppercase;
        color: #64748b;
        letter-spacing: 0.5px;
    }

    .btn-group {
        display: flex;
        background-color: #f1f5f9;
        padding: 0.25rem;
        border-radius: 8px;
        gap: 0.25rem;
        flex-wrap: wrap;
    }

    .nav-btn {
        background: transparent;
        border: none;
        padding: 0.5rem 1rem;
        font-size: 0.9rem;
        font-weight: 500;
        color: #475569;
        border-radius: 6px;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    .nav-btn:hover {
        color: #0f172a;
        background-color: rgba(255, 255, 255, 0.5);
    }

    .nav-btn.active {
        background-color: #ffffff;
        color: #0f172a;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
        font-weight: 600;
    }

    .nav-toggle-switch {
        display: flex;
        align-items: center;
        gap: 0.55rem;
        cursor: pointer;
        user-select: none;
    }

    .nav-toggle-switch input {
        display: none;
    }

    .toggle-slider {
        position: relative;
        width: 34px;
        height: 18px;
        background-color: #cbd5e1;
        border-radius: 20px;
        transition: background-color 0.2s ease;
    }

    .toggle-slider::before {
        content: "";
        position: absolute;
        width: 12px;
        height: 12px;
        border-radius: 50%;
        background-color: white;
        top: 3px;
        left: 3px;
        transition: transform 0.2s ease;
    }

    .nav-toggle-switch input:checked + .toggle-slider {
        background-color: #0f172a;
    }

    .nav-toggle-switch input:checked + .toggle-slider::before {
        transform: translateX(16px);
    }

    .toggle-caption-text {
        font-size: 0.85rem;
        font-weight: 500;
        color: #475569;
    }

    .control-fade-in {
        animation: fadeIn 0.25s ease-out forwards;
    }

    @keyframes fadeIn {
        from {
            opacity: 0;
            transform: translateY(-2px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .content-area {
        flex-grow: 1;
        padding: 2rem 0;
        display: grid;
        grid-template-columns: 1fr;
        align-items: start;
        overflow-x: hidden;
    }

    .bracket-transition-layer {
        grid-area: 1 / 1 / 2 / 2;
        width: 100%;
    }

    @media (max-width: 768px) {
        .nav-container {
            flex-direction: column;
            align-items: flex-start;
            gap: 1rem;
        }
        .nav-section {
            width: 100%;
            display: flex;
            flex-direction: column;
            align-items: flex-start;
            gap: 0.5rem;
        }
        .btn-group {
            width: 100%;
        }
    }
</style>
