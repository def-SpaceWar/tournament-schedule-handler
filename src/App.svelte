<script lang="ts">
    import { fly, fade, scale } from "svelte/transition";
    import Bracket, {
        type AgeGroup,
        type Sex,
        type Sport,
    } from "./lib/Bracket.svelte";
    import { firestore, auth } from "./lib/firebase"; // Imported auth
    import { collection, onSnapshot } from "firebase/firestore";
    // Imported Firebase Auth methods and types
    import {
        signInWithEmailAndPassword,
        signOut,
        onAuthStateChanged,
        type User,
    } from "firebase/auth";

    interface TournamentOption {
        id: string;
        label: string;
        ageGroup: AgeGroup;
        sex: Sex;
        doubleElim?: boolean;
        totalTeams: number;
    }

    const sports: Sport[] = ["soccer", "volleyball", "basketball"];

    // --- App Reactive States ---
    let currentSport = $state<Sport>("soccer"),
        availableGroups = $state<TournamentOption[]>([]),
        selectedOptionIndex = $state<number>(0),
        useColorLines = $state(false);

    // --- Admin Auth States ---
    let currentUser = $state<User | null>(null);
    let showLoginModal = $state(false);
    let email = $state("");
    let password = $state("");
    let authError = $state("");

    let currentOption = $derived(
        availableGroups[selectedOptionIndex] || availableGroups[0],
    );

    // --- Auth Observer Effect ---
    $effect(() => {
        const unsubscribe = onAuthStateChanged(auth, (user) => {
            currentUser = user;
        });
        return unsubscribe; // Auto-cleanup when component destroys
    });

    // --- Live-Synced Groups Effect ---
    // Uses onSnapshot rather than a one-time getDocs so that admin edits made
    // from within the Bracket admin panel (team_count, double_elimination, etc.)
    // are reflected here immediately - e.g. the "Chromatic Routes" toggle below
    // depends on currentOption.doubleElim staying in sync with Firestore.
    $effect(() => {
        const groupsRef = collection(
            firestore,
            "sports",
            currentSport,
            "groups",
        );

        const unsubscribe = onSnapshot(
            groupsRef,
            (snapshot) => {
                availableGroups = snapshot.docs.map((doc) => {
                    const data = doc.data();
                    return {
                        id: doc.id,
                        label: data.label || "Unnamed Group",
                        ageGroup: data.ageGroup as AgeGroup,
                        sex: data.sex as Sex,
                        doubleElim: data.double_elimination || false,
                        // team_count is sometimes stored as a string in Firestore (e.g. "0"),
                        // so coerce to a real number or "0" would otherwise be treated as truthy.
                        totalTeams: Number(data.team_count) || 0,
                    };
                });
            },
            (error) => {
                console.error(
                    `Failed to sync tournament config for ${currentSport}:`,
                    error,
                );
                availableGroups = [];
            },
        );

        return unsubscribe;
    });

    $effect(() => {
        if (selectedOptionIndex >= availableGroups.length) {
            selectedOptionIndex = 0;
        }
    });

    async function handleLogin(e: SubmitEvent) {
        e.preventDefault();
        authError = "";
        try {
            await signInWithEmailAndPassword(auth, email, password);
            showLoginModal = false;
            email = "";
            password = "";
        } catch (error: any) {
            authError = error.message.replace("Firebase: ", "");
        }
    }

    async function handleLogout() {
        try {
            await signOut(auth);
        } catch (error) {
            console.error("Logout failed:", error);
        }
    }
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

            <div class="nav-admin-section">
                {#if currentUser}
                    <div class="admin-status-badge">
                        <span class="indicator-dot"></span>
                        <span class="admin-email">{currentUser.email}</span>
                    </div>
                    <button
                        class="action-btn logout-variant"
                        onclick={handleLogout}
                    >
                        Log Out
                    </button>
                {:else}
                    <button
                        class="action-btn login-variant"
                        onclick={() => (showLoginModal = true)}
                    >
                        Admin Portal
                    </button>
                {/if}
            </div>
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
                        bracketId={currentOption?.id}
                        divisionLabel={currentOption?.label}
                        sex={currentOption?.sex}
                        ageGroup={currentOption?.ageGroup}
                        sport={currentSport}
                        totalTeams={currentOption?.totalTeams}
                        isDoubleElimination={currentOption?.doubleElim ?? false}
                        isAdmin={!!currentUser}
                        {useColorLines}
                    />
                </div>
            {/key}
        {/if}
    </main>
</div>

{#if showLoginModal}
    <div
        class="modal-backdrop"
        transition:fade={{ duration: 150 }}
        onclick={() => (showLoginModal = false)}
        role="presentation"
    >
        <div
            class="modal-surface"
            transition:scale={{ start: 0.95, duration: 150 }}
            onclick={(e) => e.stopPropagation()}
            role="dialog"
            aria-modal="true"
        >
            <div class="modal-header">
                <h3>Admin Authentication</h3>
                <p>Access control challenge for terminal operations.</p>
            </div>

            <form onsubmit={handleLogin} class="modal-form">
                {#if authError}
                    <div class="auth-error-banner">
                        {authError}
                    </div>
                {/if}

                <div class="form-field">
                    <label for="email">Admin Email Address</label>
                    <input
                        type="email"
                        id="email"
                        bind:value={email}
                        required
                        placeholder="admin@domain.com"
                    />
                </div>

                <div class="form-field">
                    <label for="password">Security Cryptographic Key</label>
                    <input
                        type="password"
                        id="password"
                        bind:value={password}
                        required
                        placeholder="••••••••"
                    />
                </div>

                <div class="form-actions">
                    <button
                        type="button"
                        class="form-btn secondary"
                        onclick={() => (showLoginModal = false)}
                    >
                        Cancel
                    </button>
                    <button type="submit" class="form-btn primary">
                        Verify Identity
                    </button>
                </div>
            </form>
        </div>
    </div>
{/if}

<style>
    :global(body) {
        margin: 0;
        background-color: #f8fafc;
    }

    .dashboard-layout {
        display: flex;
        flex-direction: column;
        height: 100vh;
        overflow: hidden;
    }

    .top-nav {
        flex-shrink: 0;
        background-color: #ffffff;
        border-bottom: 1px solid #e2e8f0;
        padding: 1rem 2rem;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
        z-index: 10;
    }

    .nav-container {
        max-width: 1400px;
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

    /* Pulls admin segment out out line elements to right viewport edge */
    .nav-admin-section {
        margin-left: auto;
        display: flex;
        align-items: center;
        gap: 1rem;
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

    /* --- Specialized Admin Navigation Utilities --- */
    .action-btn {
        border: 1px solid transparent;
        padding: 0.5rem 1rem;
        font-size: 0.85rem;
        font-weight: 600;
        border-radius: 6px;
        cursor: pointer;
        transition: all 0.15s ease;
    }

    .action-btn.login-variant {
        background-color: #0f172a;
        color: #ffffff;
    }

    .action-btn.login-variant:hover {
        background-color: #1e293b;
    }

    .action-btn.logout-variant {
        background-color: transparent;
        border-color: #e2e8f0;
        color: #64748b;
    }

    .action-btn.logout-variant:hover {
        background-color: #f1f5f9;
        color: #0f172a;
    }

    .admin-status-badge {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        background-color: #f0fdf4;
        border: 1px solid #bbf7d0;
        padding: 0.4rem 0.75rem;
        border-radius: 6px;
    }

    .indicator-dot {
        width: 6px;
        height: 6px;
        background-color: #22c55e;
        border-radius: 50%;
    }

    .admin-email {
        font-size: 0.8rem;
        font-weight: 500;
        color: #166534;
    }

    /* --- Control Toggles --- */
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

    /* --- Content Frame Matrix --- */
    .content-area {
        flex-grow: 1;
        display: grid;
        grid-template-columns: 1fr;
        align-items: stretch;
        padding: 1.5rem;
        overflow: hidden;
        min-height: 0;
    }

    .bracket-transition-layer {
        grid-area: 1 / 1 / 2 / 2;
        width: 100%;
        height: 100%;
        overflow: auto;
        background: #ffffff;
        border: 1px solid #e2e8f0;
        border-radius: 12px;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.02);
    }

    /* --- Modal Structure Core Overlay --- */
    .modal-backdrop {
        position: fixed;
        inset: 0;
        background-color: rgba(15, 23, 42, 0.3);
        backdrop-filter: blur(4px);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 100;
        padding: 1rem;
    }

    .modal-surface {
        background-color: #ffffff;
        border: 1px solid #e2e8f0;
        border-radius: 12px;
        width: 100%;
        max-width: 400px;
        box-shadow:
            0 20px 25px -5px rgba(0, 0, 0, 0.1),
            0 10px 10px -5px rgba(0, 0, 0, 0.04);
        overflow: hidden;
    }

    .modal-header {
        padding: 1.5rem 1.5rem 1rem 1.5rem;
        border-bottom: 1px solid #f1f5f9;
    }

    .modal-header h3 {
        margin: 0;
        font-size: 1.2rem;
        font-weight: 700;
        color: #0f172a;
    }

    .modal-header p {
        margin: 0.25rem 0 0 0;
        font-size: 0.85rem;
        color: #64748b;
    }

    .modal-form {
        padding: 1.5rem;
        display: flex;
        flex-direction: column;
        gap: 1.25rem;
    }

    .auth-error-banner {
        background-color: #fef2f2;
        border: 1px solid #fca5a5;
        color: #991b1b;
        padding: 0.75rem;
        border-radius: 6px;
        font-size: 0.8rem;
        font-weight: 500;
    }

    .form-field {
        display: flex;
        flex-direction: column;
        gap: 0.4rem;
    }

    .form-field label {
        font-size: 0.8rem;
        font-weight: 600;
        color: #475569;
        text-transform: uppercase;
        letter-spacing: 0.3px;
    }

    .form-field input {
        padding: 0.6rem 0.75rem;
        font-size: 0.95rem;
        border: 1px solid #cbd5e1;
        border-radius: 6px;
        outline: none;
        color: #0f172a;
        transition: border-color 0.15s ease;
    }

    .form-field input:focus {
        border-color: #0f172a;
    }

    .form-actions {
        display: flex;
        justify-content: flex-end;
        gap: 0.75rem;
        margin-top: 0.5rem;
    }

    .form-btn {
        padding: 0.55rem 1.25rem;
        font-size: 0.9rem;
        font-weight: 600;
        border-radius: 6px;
        cursor: pointer;
        border: 1px solid transparent;
        transition: all 0.15s ease;
    }

    .form-btn.primary {
        background-color: #0f172a;
        color: #ffffff;
    }

    .form-btn.primary:hover {
        background-color: #1e293b;
    }

    .form-btn.secondary {
        background-color: transparent;
        border-color: #e2e8f0;
        color: #475569;
    }

    .form-btn.secondary:hover {
        background-color: #f1f5f9;
        color: #0f172a;
    }

    @media (max-width: 768px) {
        .top-nav {
            padding: 0.75rem 1rem;
        }
        .dashboard-layout {
            height: auto;
            overflow: auto;
        }
        .content-area {
            overflow: visible;
            height: auto;
            padding: 0.75rem;
        }
        .bracket-transition-layer {
            overflow: auto;
            height: max-content;
        }
        .nav-container {
            flex-direction: column;
            align-items: flex-start;
            gap: 0.75rem;
        }
        .nav-section {
            width: 100%;
            flex-wrap: wrap;
        }
        .nav-admin-section {
            width: 100%;
            margin-left: 0;
            padding-top: 0.5rem;
            border-top: 1px solid #f1f5f9;
            flex-direction: row;
            flex-wrap: wrap;
            gap: 0.5rem;
        }
        .btn-group {
            flex: 1;
        }
        .action-btn {
            flex: 1;
            text-align: center;
        }
        .admin-status-badge {
            width: 100%;
        }
    }

    @media (max-width: 480px) {
        .nav-btn {
            padding: 0.4rem 0.6rem;
            font-size: 0.82rem;
        }
        .modal-surface {
            border-radius: 10px;
        }
        .modal-form {
            padding: 1rem;
        }
        .form-actions {
            flex-direction: column-reverse;
        }
        .form-btn {
            width: 100%;
            text-align: center;
        }
    }
</style>
