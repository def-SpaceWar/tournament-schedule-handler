<script lang="ts">
    import { firestore } from "./firebase";
    import {
        doc,
        collection,
        onSnapshot,
        updateDoc,
        addDoc,
        deleteDoc,
        writeBatch,
        setDoc,
    } from "firebase/firestore";

    export type Sport = "soccer" | "basketball" | "volleyball";
    export type AgeGroup = "open" | "under10" | "under14" | "under18";
    export type Sex = "male" | "female" | "both";

    // Component Props using Svelte 5 Runes
    let {
        bracketId, // Document ID passed from parent App.svelte
        divisionLabel,
        sex,
        ageGroup,
        sport,
        totalTeams,
        isDoubleElimination = true,
        useColorLines = false,
        isAdmin = false,
    }: {
        bracketId: string;
        divisionLabel: string;
        sex: Sex;
        ageGroup: AgeGroup;
        sport: Sport;
        totalTeams: number;
        isDoubleElimination?: boolean;
        useColorLines?: boolean;
        isAdmin?: boolean;
    } = $props();

    interface Match {
        id: string;
        p1: string;
        p2: string;
    }

    interface Connection {
        fromId: string;
        type: "winner" | "loser";
        toId: string;
        toSlot: "p1" | "p2";
    }

    interface Team {
        id: string;
        name: string;
        players: string[];
        logoUrl: string;
    }

    interface FirestoreMatch {
        id: string;
        p1: string | null;
        p2: string | null;
        winner: string | null;
        nextMatchId: string | null;
        nextSlot: "p1" | "p2" | null;
        loserMatchId: string | null;
        loserSlot: "p1" | "p2" | null;
        stadiumId?: string | null;
        stadium?: string | null; // legacy plain text fallback
        isLive?: boolean;
        p1Score?: number | null;
        p2Score?: number | null;
        startTime?: string | null; // ISO datetime string
    }

    interface Stadium {
        id: string;
        name: string;
        livestreamUrl?: string | null;
    }

    // --- Admin & Bracket Management State ---
    let teamCount = $state(totalTeams);
    let doubleElimEnabled = $state(isDoubleElimination);
    let newTeamName = $state("");
    let newTeamLogoUrl = $state("");
    let newTeamPlayersInput = $state("");

    // Inline team-roster editor state (which team card is expanded for editing)
    let editingTeamId = $state<string | null>(null);
    let editTeamName = $state("");
    let editTeamLogoUrl = $state("");
    let editTeamPlayersInput = $state("");

    // Match-details modal state (stadium + live status + scores)
    let activeDetailsMatchId = $state<string | null>(null);
    let detailsStadiumInput = $state("");
    let detailsStadiumIdInput = $state<string>("");
    let detailsIsLiveInput = $state(false);
    let detailsP1ScoreInput = $state("");
    let detailsP2ScoreInput = $state("");
    let detailsStartTimeInput = $state("");
    let detailsWinnerInput = $state<string>("");

    // Stadium management state
    let dbStadiums = $state<Stadium[]>([]);
    let showStadiumManager = $state(false);
    let newStadiumName = $state("");
    let newStadiumUrl = $state("");
    let editingStadiumId = $state<string | null>(null);
    let editStadiumName = $state("");
    let editStadiumUrl = $state("");

    // Real-time synced arrays from Firestore
    let dbTeams = $state<Team[]>([]);
    let dbMatches = $state<Record<string, FirestoreMatch>>({});

    let containerEl = $state<HTMLElement | null>(null),
        svgPaths = $state<{ d: string; color: string }[]>([]),
        championshipTop = $state<number>(0);

    const NODE_HEIGHT = 80,
        MIN_VERTICAL_GAP = 48,
        TOTAL_NODE_SPACING = NODE_HEIGHT + MIN_VERTICAL_GAP;

    // --- Real-Time Synchronizer Effect ---
    $effect(() => {
        if (!bracketId || !sport) return;

        // 1. Sync the group's own config doc
        const unsubGroup = onSnapshot(
            doc(firestore, "sports", sport, "groups", bracketId),
            (snap) => {
                if (!snap.exists()) return;
                const data = snap.data();
                if (data.team_count !== undefined) {
                    teamCount = Number(data.team_count) || 0;
                }
                if (data.double_elimination !== undefined) {
                    doubleElimEnabled = !!data.double_elimination;
                }
            },
        );

        // 2. Sync subcollection of teams, nested under the group document
        const unsubTeams = onSnapshot(
            collection(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "teams",
            ),
            (snap) => {
                dbTeams = snap.docs.map((d) => {
                    const data = d.data();
                    return {
                        id: d.id,
                        name: data.name || "Unnamed Team",
                        players: Array.isArray(data.players)
                            ? data.players
                            : [],
                        logoUrl: data.logoUrl || "",
                    };
                });
            },
        );

        // 3. Sync subcollection of match documents
        const unsubMatches = onSnapshot(
            collection(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "bracket",
            ),
            (snap) => {
                const matchMap: Record<string, FirestoreMatch> = {};
                snap.docs.forEach((d) => {
                    matchMap[d.id] = {
                        id: d.id,
                        ...d.data(),
                    } as FirestoreMatch;
                });
                dbMatches = matchMap;
            },
        );

        // 4. Sync stadiums subcollection
        const unsubStadiums = onSnapshot(
            collection(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "stadiums",
            ),
            (snap) => {
                dbStadiums = snap.docs.map((d) => ({
                    id: d.id,
                    name: d.data().name || "Unnamed Stadium",
                    livestreamUrl: d.data().livestreamUrl || null,
                }));
            },
        );

        return () => {
            unsubGroup();
            unsubTeams();
            unsubMatches();
            unsubStadiums();
        };
    });

    // --- Dynamic Bracket Layout and Name Propagation Engine ---
    let bracketData = $derived.by(() => {
        const wbRounds: Match[][] = [],
            lbRounds: Match[][] = [],
            connections: Connection[] = [],
            wbLosersByRound: { name: string; sourceId: string }[][] = [];

        function registerConnection(
            fromId: string,
            toId: string,
            toSlot: "p1" | "p2",
        ) {
            const isLoserRoute =
                fromId.startsWith("WB") && toId.startsWith("LB");
            connections.push({
                fromId,
                type: isLoserRoute ? "loser" : "winner",
                toId,
                toSlot,
            });
        }

        const teamsCount = teamCount || 0;
        if (teamsCount < 2) return { wb: [], lb: [], connections };

        const nextPower = Math.pow(2, Math.ceil(Math.log2(teamsCount))),
            numMatchesR1 = nextPower / 2,
            numDuoMatches = teamsCount - numMatchesR1;

        let round1Matches: Match[] = [],
            nextWB: { name: string; sourceId: string }[] = [],
            round1Losers: { name: string; sourceId: string }[] = [],
            teamIdx = 0;

        // Round 1 Generation
        for (let m = 0; m < numMatchesR1; m++) {
            let shortMatchId = `WB-R1-M${m + 1}`;
            const dbM = dbMatches[shortMatchId];

            let p1 = dbM?.p1
                ? dbTeams.find((t) => t.id === dbM.p1)?.name || "Unknown"
                : `Team ${teamIdx + 1}`;
            let p2 = "BYE";
            if (m < numDuoMatches) {
                p2 = dbM?.p2
                    ? dbTeams.find((t) => t.id === dbM.p2)?.name || "Unknown"
                    : `Team ${teamIdx + 2}`;
                teamIdx += 2;
            } else {
                if (dbM?.p2)
                    p2 =
                        dbM.p2 === "BYE"
                            ? "BYE"
                            : dbTeams.find((t) => t.id === dbM.p2)?.name ||
                              "Unknown";
                teamIdx += 1;
            }

            round1Matches.push({ id: shortMatchId, p1, p2 });

            if (p2 === "BYE") {
                nextWB.push({ name: p1, sourceId: shortMatchId });
            } else {
                let winnerName = `Winner ${shortMatchId}`;
                if (dbM?.winner) {
                    winnerName =
                        dbTeams.find((t) => t.id === dbM.winner)?.name ||
                        "Unknown";
                }
                nextWB.push({ name: winnerName, sourceId: shortMatchId });

                let loserName = `Loser ${shortMatchId}`;
                if (dbM?.winner) {
                    const loserId = dbM.winner === dbM.p1 ? dbM.p2 : dbM.p1;
                    if (loserId)
                        loserName =
                            dbTeams.find((t) => t.id === loserId)?.name ||
                            "Unknown";
                }
                round1Losers.push({ name: loserName, sourceId: shortMatchId });
            }
        }
        wbRounds.push(round1Matches);
        wbLosersByRound.push(round1Losers);

        // Subsequent Winners Bracket Rounds
        let currentWB = [...nextWB],
            rIdx = 2;
        while (currentWB.length > 1) {
            let roundMatches: Match[] = [],
                nextWBTemp: typeof currentWB = [],
                roundLosers: { name: string; sourceId: string }[] = [];

            for (let i = 0; i < currentWB.length; i += 2) {
                let shortMatchId = `WB-R${rIdx}-M${Math.floor(i / 2) + 1}`;
                const dbM = dbMatches[shortMatchId];

                let p1 = dbM?.p1
                    ? dbTeams.find((t) => t.id === dbM.p1)?.name || "Unknown"
                    : currentWB[i].name;
                let p2 = dbM?.p2
                    ? dbTeams.find((t) => t.id === dbM.p2)?.name || "Unknown"
                    : currentWB[i + 1].name;

                roundMatches.push({ id: shortMatchId, p1, p2 });

                if (currentWB[i].sourceId)
                    registerConnection(
                        currentWB[i].sourceId,
                        shortMatchId,
                        "p1",
                    );
                if (currentWB[i + 1].sourceId)
                    registerConnection(
                        currentWB[i + 1].sourceId,
                        shortMatchId,
                        "p2",
                    );

                let winnerName = `Winner ${shortMatchId}`;
                if (dbM?.winner) {
                    winnerName =
                        dbTeams.find((t) => t.id === dbM.winner)?.name ||
                        "Unknown";
                }
                nextWBTemp.push({ name: winnerName, sourceId: shortMatchId });

                let loserName = `Loser ${shortMatchId}`;
                if (dbM?.winner) {
                    const loserId = dbM.winner === dbM.p1 ? dbM.p2 : dbM.p1;
                    if (loserId)
                        loserName =
                            dbTeams.find((t) => t.id === loserId)?.name ||
                            "Unknown";
                }
                roundLosers.push({ name: loserName, sourceId: shortMatchId });
            }
            wbRounds.push(roundMatches);
            wbLosersByRound.push(roundLosers);
            currentWB = nextWBTemp;
            rIdx++;
        }

        if (doubleElimEnabled && wbRounds.length > 0) {
            let lastWB = wbRounds[wbRounds.length - 1][0];
            if (lastWB) registerConnection(lastWB.id, "CHAMPIONSHIP", "p1");
        }

        // Losers Bracket Progression Setup
        if (doubleElimEnabled && wbLosersByRound.length > 0) {
            let lbSurvivors: { name: string; sourceId: string }[] = [],
                lbRoundIdx = 1;
            for (let w = 0; w < wbLosersByRound.length; w++) {
                const freshLosers = wbLosersByRound[w] || [];

                if (w === 0) {
                    let numLBMatches = wbRounds[0].length / 2,
                        roundMatches: Match[] = [],
                        nextSurvivors: typeof lbSurvivors = [];
                    for (let i = 0; i < numLBMatches; i++) {
                        let matchId = `LB-R${lbRoundIdx}-M${i + 1}`,
                            p1Obj = freshLosers[i * 2],
                            p2Obj = freshLosers[i * 2 + 1];
                        const dbM = dbMatches[matchId];

                        const p1 = dbM?.p1
                            ? dbTeams.find((t) => t.id === dbM.p1)?.name ||
                              "Unknown"
                            : p1Obj
                              ? p1Obj.name
                              : "BYE";
                        const p2 = dbM?.p2
                            ? dbTeams.find((t) => t.id === dbM.p2)?.name ||
                              "Unknown"
                            : p2Obj
                              ? p2Obj.name
                              : "BYE";

                        if (p1 === "BYE" && p2 === "BYE") continue;
                        roundMatches.push({ id: matchId, p1, p2 });

                        let winnerName = `Winner ${matchId}`;
                        if (dbM?.winner) {
                            winnerName =
                                dbTeams.find((t) => t.id === dbM.winner)
                                    ?.name || "Unknown";
                        }
                        nextSurvivors.push({
                            name: winnerName,
                            sourceId: matchId,
                        });

                        if (p1Obj?.sourceId)
                            registerConnection(p1Obj.sourceId, matchId, "p1");
                        if (p2Obj?.sourceId)
                            registerConnection(p2Obj.sourceId, matchId, "p2");
                    }
                    if (roundMatches.length > 0) {
                        lbRounds.push(roundMatches);
                        lbRoundIdx++;
                    }
                    lbSurvivors = nextSurvivors;
                } else {
                    if (
                        lbSurvivors.length > freshLosers.length &&
                        freshLosers.length > 0
                    ) {
                        let stageAMatches: Match[] = [],
                            stageANext: typeof lbSurvivors = [],
                            numMatches = lbSurvivors.length / 2;
                        for (let i = 0; i < numMatches; i++) {
                            let matchId = `LB-R${lbRoundIdx}A-M${i + 1}`,
                                p1Obj = lbSurvivors[i * 2],
                                p2Obj = lbSurvivors[i * 2 + 1];
                            const dbM = dbMatches[matchId];

                            const p1 = dbM?.p1
                                ? dbTeams.find((t) => t.id === dbM.p1)?.name ||
                                  "Unknown"
                                : p1Obj
                                  ? p1Obj.name
                                  : "BYE";
                            const p2 = dbM?.p2
                                ? dbTeams.find((t) => t.id === dbM.p2)?.name ||
                                  "Unknown"
                                : p2Obj
                                  ? p2Obj.name
                                  : "BYE";

                            if (p1 === "BYE" && p2 === "BYE") continue;
                            stageAMatches.push({ id: matchId, p1, p2 });

                            let winnerName = `Winner ${matchId}`;
                            if (dbM?.winner) {
                                winnerName =
                                    dbTeams.find((t) => t.id === dbM.winner)
                                        ?.name || "Unknown";
                            }
                            stageANext.push({
                                name: winnerName,
                                sourceId: matchId,
                            });

                            if (p1Obj?.sourceId)
                                registerConnection(
                                    p1Obj.sourceId,
                                    matchId,
                                    "p1",
                                );
                            if (p2Obj?.sourceId)
                                registerConnection(
                                    p2Obj.sourceId,
                                    matchId,
                                    "p2",
                                );
                        }
                        if (stageAMatches.length > 0)
                            lbRounds.push(stageAMatches);
                        lbSurvivors = stageANext;
                    }

                    let stageBMatches: Match[] = [],
                        stageBNext: typeof lbSurvivors = [],
                        numBMatches = freshLosers.length;
                    for (let i = 0; i < numBMatches; i++) {
                        let matchId = `LB-R${lbRoundIdx}B-M${i + 1}`,
                            p1Obj = lbSurvivors[i],
                            p2Obj = freshLosers[i];
                        const dbM = dbMatches[matchId];

                        const p1 = dbM?.p1
                            ? dbTeams.find((t) => t.id === dbM.p1)?.name ||
                              "Unknown"
                            : p1Obj
                              ? p1Obj.name
                              : "BYE";
                        const p2 = dbM?.p2
                            ? dbTeams.find((t) => t.id === dbM.p2)?.name ||
                              "Unknown"
                            : p2Obj
                              ? p2Obj.name
                              : "BYE";

                        if (p1 === "BYE" && p2 === "BYE") continue;
                        stageBMatches.push({ id: matchId, p1, p2 });

                        let winnerName = `Winner ${matchId}`;
                        if (dbM?.winner) {
                            winnerName =
                                dbTeams.find((t) => t.id === dbM.winner)
                                    ?.name || "Unknown";
                        }
                        stageBNext.push({
                            name: winnerName,
                            sourceId: matchId,
                        });

                        if (p1Obj?.sourceId)
                            registerConnection(p1Obj.sourceId, matchId, "p1");
                        if (p2Obj?.sourceId)
                            registerConnection(p2Obj.sourceId, matchId, "p2");
                    }
                    if (stageBMatches.length > 0) {
                        lbRounds.push(stageBMatches);
                        lbRoundIdx++;
                    }
                    lbSurvivors = stageBNext;
                }
            }

            if (lbRounds.length > 0) {
                let lastLB = lbRounds[lbRounds.length - 1][0];
                if (lastLB) registerConnection(lastLB.id, "CHAMPIONSHIP", "p2");
            }
        }

        return { wb: wbRounds, lb: lbRounds, connections };
    });

    // --- Node Coordinates Calculation Layout Engine ---
    let yPositions = $derived.by(() => {
        const pos: Record<string, number> = {};
        const wbR1 = bracketData.wb[0] || [];
        wbR1.forEach((match, i) => (pos[match.id] = i * TOTAL_NODE_SPACING));

        for (let r = 1; r < bracketData.wb.length; r++) {
            bracketData.wb[r].forEach((match) => {
                let parents = bracketData.connections
                        .filter((c) => c.toId === match.id)
                        .map((c) => c.fromId),
                    validParentY = parents
                        .map((pId) => pos[pId])
                        .filter((y) => y !== undefined);

                if (validParentY.length > 0)
                    pos[match.id] =
                        validParentY.reduce((a, b) => a + b, 0) /
                        validParentY.length;
                else pos[match.id] = 0;
            });

            let roundMatches = [...bracketData.wb[r]];
            roundMatches.sort((a, b) => pos[a.id] - pos[b.id]);
            for (let i = 1; i < roundMatches.length; i++) {
                let prevId = roundMatches[i - 1].id,
                    currId = roundMatches[i].id;
                if (pos[currId] < pos[prevId] + TOTAL_NODE_SPACING)
                    pos[currId] = pos[prevId] + TOTAL_NODE_SPACING;
            }
        }

        if (doubleElimEnabled) {
            let lbR1 = bracketData.lb[0] || [];
            lbR1.forEach(
                (match, i) => (pos[match.id] = i * TOTAL_NODE_SPACING),
            );
            for (let r = 1; r < bracketData.lb.length; r++) {
                bracketData.lb[r].forEach((match) => {
                    let parents = bracketData.connections
                            .filter((c) => c.toId === match.id)
                            .map((c) => c.fromId),
                        validParentY = parents
                            .map((pId) => pos[pId])
                            .filter((y) => y !== undefined);

                    if (validParentY.length > 0)
                        pos[match.id] =
                            validParentY.reduce((a, b) => a + b, 0) /
                            validParentY.length;
                    else {
                        let idx = bracketData.lb[r].findIndex(
                            (m) => m.id === match.id,
                        );
                        pos[match.id] = idx * TOTAL_NODE_SPACING;
                    }
                });
                let roundMatches = [...bracketData.lb[r]];
                roundMatches.sort((a, b) => pos[a.id] - pos[b.id]);
                for (let i = 1; i < roundMatches.length; i++) {
                    let prevId = roundMatches[i - 1].id,
                        currId = roundMatches[i].id;
                    if (pos[currId] < pos[prevId] + TOTAL_NODE_SPACING)
                        pos[currId] = pos[prevId] + TOTAL_NODE_SPACING;
                }
            }
        }
        pos["CHAMPIONSHIP"] = championshipTop;
        return pos;
    });

    let laneHeights = $derived.by(() => {
        let maxWB = Math.max(
                ...bracketData.wb.flat().map((m) => yPositions[m.id] || 0),
                0,
            ),
            maxLB = Math.max(
                ...bracketData.lb.flat().map((m) => yPositions[m.id] || 0),
                0,
            );
        return { wb: maxWB + NODE_HEIGHT + 20, lb: maxLB + NODE_HEIGHT + 20 };
    });

    // --- Action Methods linked to Firestore ---
    function teamLogoFor(teamId: string | null | undefined): string {
        if (!teamId || teamId === "BYE") return "";
        return dbTeams.find((t) => t.id === teamId)?.logoUrl || "";
    }

    function parsePlayersInput(raw: string): string[] {
        return raw
            .split(",")
            .map((s) => s.trim())
            .filter((s) => s.length > 0);
    }

    async function handleAddTeam() {
        if (newTeamName.trim() !== "" && bracketId && sport) {
            await addDoc(
                collection(
                    firestore,
                    "sports",
                    sport,
                    "groups",
                    bracketId,
                    "teams",
                ),
                {
                    name: newTeamName.trim(),
                    logoUrl: newTeamLogoUrl.trim(),
                    players: parsePlayersInput(newTeamPlayersInput),
                },
            );
            newTeamName = "";
            newTeamLogoUrl = "";
            newTeamPlayersInput = "";
        }
    }

    function startEditTeam(team: Team) {
        editingTeamId = team.id;
        editTeamName = team.name;
        editTeamLogoUrl = team.logoUrl;
        editTeamPlayersInput = team.players.join(", ");
    }

    function cancelEditTeam() {
        editingTeamId = null;
    }

    async function handleSaveTeamEdits() {
        if (!isAdmin || !bracketId || !sport || !editingTeamId) return;
        const teamRef = doc(
            firestore,
            "sports",
            sport,
            "groups",
            bracketId,
            "teams",
            editingTeamId,
        );
        await updateDoc(teamRef, {
            name: editTeamName.trim() || "Unnamed Team",
            logoUrl: editTeamLogoUrl.trim(),
            players: parsePlayersInput(editTeamPlayersInput),
        });
        editingTeamId = null;
    }

    async function handleDeleteTeam(teamId: string) {
        if (!isAdmin || !bracketId || !sport) return;
        if (!confirm("Remove this team? This cannot be undone.")) return;
        await deleteDoc(
            doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "teams",
                teamId,
            ),
        );
        if (editingTeamId === teamId) editingTeamId = null;
    }

    async function handleUpdateTeamCount(newCount: number) {
        if (!isAdmin || !bracketId || !sport) return;
        await updateDoc(doc(firestore, "sports", sport, "groups", bracketId), {
            team_count: newCount,
        });
    }

    async function handleUpdateDoubleElimination(enabled: boolean) {
        if (!isAdmin || !bracketId || !sport) return;
        await updateDoc(doc(firestore, "sports", sport, "groups", bracketId), {
            double_elimination: enabled,
        });
    }

    async function handleUpdateMatchSlot(
        matchId: string,
        slot: "p1" | "p2",
        teamId: string,
    ) {
        if (!isAdmin || !bracketId || !sport) return;
        const matchRef = doc(
            firestore,
            "sports",
            sport,
            "groups",
            bracketId,
            "bracket",
            matchId,
        );

        await setDoc(
            matchRef,
            {
                id: matchId,
                [slot]: teamId === "" ? null : teamId,
            },
            { merge: true },
        );
    }

    async function handleDeclareWinner(matchId: string, winnerTeamId: string) {
        if (
            !isAdmin ||
            !bracketId ||
            !sport ||
            !winnerTeamId ||
            winnerTeamId === "BYE"
        )
            return;

        const currentMatch = dbMatches[matchId];
        if (!currentMatch) return;

        const loserTeamId =
            winnerTeamId === currentMatch.p1
                ? currentMatch.p2
                : currentMatch.p1;

        const matchRef = doc(
            firestore,
            "sports",
            sport,
            "groups",
            bracketId,
            "bracket",
            matchId,
        );
        await updateDoc(matchRef, { winner: winnerTeamId });

        if (currentMatch.nextMatchId && currentMatch.nextSlot) {
            const nextMatchRef = doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "bracket",
                currentMatch.nextMatchId,
            );
            await updateDoc(nextMatchRef, {
                [currentMatch.nextSlot]: winnerTeamId,
            });
        }

        if (
            currentMatch.loserMatchId &&
            currentMatch.loserSlot &&
            loserTeamId &&
            loserTeamId !== "BYE"
        ) {
            const loserMatchRef = doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "bracket",
                currentMatch.loserMatchId,
            );
            await updateDoc(loserMatchRef, {
                [currentMatch.loserSlot]: loserTeamId,
            });
        }
    }

    // --- Match Details Modal Core Config Management ---
    function openMatchDetails(matchId: string) {
        const dbM = dbMatches[matchId];
        activeDetailsMatchId = matchId;
        detailsStadiumInput = dbM?.stadium || "";
        detailsStadiumIdInput = dbM?.stadiumId || "";
        detailsIsLiveInput = !!dbM?.isLive;
        detailsWinnerInput = dbM?.winner || "";
        detailsP1ScoreInput =
            dbM?.p1Score !== undefined && dbM?.p1Score !== null
                ? String(dbM.p1Score)
                : "";
        detailsP2ScoreInput =
            dbM?.p2Score !== undefined && dbM?.p2Score !== null
                ? String(dbM.p2Score)
                : "";
        // Convert stored ISO string to local datetime-local format
        if (dbM?.startTime) {
            const d = new Date(dbM.startTime);
            const pad = (n: number) => String(n).padStart(2, "0");
            detailsStartTimeInput = `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}T${pad(d.getHours())}:${pad(d.getMinutes())}`;
        } else {
            detailsStartTimeInput = "";
        }
    }

    function closeMatchDetails() {
        activeDetailsMatchId = null;
    }

    async function handleSaveMatchDetails() {
        if (!isAdmin || !bracketId || !sport || !activeDetailsMatchId) return;
        const matchId = activeDetailsMatchId;
        const currentMatch = dbMatches[matchId];
        const matchRef = doc(
            firestore,
            "sports",
            sport,
            "groups",
            bracketId,
            "bracket",
            matchId,
        );
        const selectedStadium = dbStadiums.find(
            (s) => s.id === detailsStadiumIdInput,
        );

        const newWinner = detailsWinnerInput || null;
        const oldWinner = currentMatch?.winner || null;

        // Persist core match fields
        await updateDoc(matchRef, {
            stadiumId: detailsStadiumIdInput || null,
            stadium:
                selectedStadium?.name || detailsStadiumInput.trim() || null,
            isLive: detailsIsLiveInput,
            startTime: detailsStartTimeInput
                ? new Date(detailsStartTimeInput).toISOString()
                : null,
            winner: newWinner,
            p1Score:
                detailsP1ScoreInput.trim() !== "" &&
                !isNaN(Number(detailsP1ScoreInput))
                    ? Number(detailsP1ScoreInput)
                    : null,
            p2Score:
                detailsP2ScoreInput.trim() !== "" &&
                !isNaN(Number(detailsP2ScoreInput))
                    ? Number(detailsP2ScoreInput)
                    : null,
        });

        // Propagate winner change to next/loser match slots
        if (newWinner !== oldWinner && currentMatch) {
            const loserTeamId =
                newWinner === currentMatch.p1
                    ? currentMatch.p2
                    : currentMatch.p1;

            // Clear slots set by old winner propagation
            if (
                oldWinner &&
                currentMatch.nextMatchId &&
                currentMatch.nextSlot
            ) {
                const nextRef = doc(
                    firestore,
                    "sports",
                    sport,
                    "groups",
                    bracketId,
                    "bracket",
                    currentMatch.nextMatchId,
                );
                await updateDoc(nextRef, { [currentMatch.nextSlot]: null });
            }
            if (
                oldWinner &&
                currentMatch.loserMatchId &&
                currentMatch.loserSlot
            ) {
                const loserRef = doc(
                    firestore,
                    "sports",
                    sport,
                    "groups",
                    bracketId,
                    "bracket",
                    currentMatch.loserMatchId,
                );
                await updateDoc(loserRef, { [currentMatch.loserSlot]: null });
            }

            // Set new propagation
            if (newWinner) {
                if (currentMatch.nextMatchId && currentMatch.nextSlot) {
                    const nextRef = doc(
                        firestore,
                        "sports",
                        sport,
                        "groups",
                        bracketId,
                        "bracket",
                        currentMatch.nextMatchId,
                    );
                    await updateDoc(nextRef, {
                        [currentMatch.nextSlot]: newWinner,
                    });
                }
                if (
                    currentMatch.loserMatchId &&
                    currentMatch.loserSlot &&
                    loserTeamId &&
                    loserTeamId !== "BYE"
                ) {
                    const loserRef = doc(
                        firestore,
                        "sports",
                        sport,
                        "groups",
                        bracketId,
                        "bracket",
                        currentMatch.loserMatchId,
                    );
                    await updateDoc(loserRef, {
                        [currentMatch.loserSlot]: loserTeamId,
                    });
                }
            }
        }

        activeDetailsMatchId = null;
    }

    // --- Stadium CRUD ---
    async function handleAddStadium() {
        if (!newStadiumName.trim() || !bracketId || !sport) return;
        await addDoc(
            collection(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "stadiums",
            ),
            {
                name: newStadiumName.trim(),
                livestreamUrl: newStadiumUrl.trim() || null,
            },
        );
        newStadiumName = "";
        newStadiumUrl = "";
    }

    function startEditStadium(s: Stadium) {
        editingStadiumId = s.id;
        editStadiumName = s.name;
        editStadiumUrl = s.livestreamUrl || "";
    }

    async function handleSaveStadiumEdits() {
        if (!editingStadiumId || !bracketId || !sport) return;
        await updateDoc(
            doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "stadiums",
                editingStadiumId,
            ),
            {
                name: editStadiumName.trim() || "Unnamed",
                livestreamUrl: editStadiumUrl.trim() || null,
            },
        );
        editingStadiumId = null;
    }

    async function handleDeleteStadium(id: string) {
        if (!bracketId || !sport) return;
        if (!confirm("Delete this stadium?")) return;
        await deleteDoc(
            doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "stadiums",
                id,
            ),
        );
    }

    // --- Start time helpers ---
    function getStartTimeLabel(
        match: FirestoreMatch | undefined,
    ): string | null {
        if (!match?.startTime) return null;
        const start = new Date(match.startTime);
        const now = new Date();
        const diffMs = start.getTime() - now.getTime();
        const diffMins = Math.round(diffMs / 60000);
        if (diffMins > 0 && diffMins <= 60) {
            return `Starting in ${diffMins} min${diffMins === 1 ? "" : "s"}`;
        }
        if (diffMins <= 0 && diffMins > -180) {
            return "In progress";
        }
        return start.toLocaleString([], {
            month: "short",
            day: "numeric",
            hour: "2-digit",
            minute: "2-digit",
        });
    }

    function getLivestreamForMatch(
        match: FirestoreMatch | undefined,
    ): string | null {
        if (!match?.stadiumId) return null;
        const stadium = dbStadiums.find((s) => s.id === match.stadiumId);
        return stadium?.livestreamUrl || null;
    }

    async function handleAutoCreateBracket() {
        if (!isAdmin || !bracketId || !sport) return;

        const batch = writeBatch(firestore);
        const allMatches = [...bracketData.wb.flat(), ...bracketData.lb.flat()];

        if (doubleElimEnabled) {
            allMatches.push({ id: "CHAMPIONSHIP", p1: "", p2: "" });
        }

        let teamIdx = 0;

        for (const match of allMatches) {
            const matchRef = doc(
                firestore,
                "sports",
                sport,
                "groups",
                bracketId,
                "bracket",
                match.id,
            );
            const winnerRoute = bracketData.connections.find(
                (c) => c.fromId === match.id && c.type === "winner",
            );
            const loserRoute = bracketData.connections.find(
                (c) => c.fromId === match.id && c.type === "loser",
            );

            const existingMatch = dbMatches[match.id];

            const routingFields = {
                nextMatchId: winnerRoute ? winnerRoute.toId : null,
                nextSlot: winnerRoute ? winnerRoute.toSlot : null,
                loserMatchId: loserRoute ? loserRoute.toId : null,
                loserSlot: loserRoute ? loserRoute.toSlot : null,
            };

            if (existingMatch) {
                batch.set(matchRef, routingFields, { merge: true });
                continue;
            }

            let initialP1 = null;
            let initialP2 = null;

            if (match.id.startsWith("WB-R1")) {
                initialP1 = dbTeams[teamIdx]?.id || null;
                const nextPower = Math.pow(2, Math.ceil(Math.log2(teamCount)));
                const numDuoMatches = teamCount - nextPower / 2;
                const currentMatchIndex = parseInt(match.id.split("-M")[1]) - 1;

                if (currentMatchIndex < numDuoMatches) {
                    initialP2 = dbTeams[teamIdx + 1]?.id || null;
                    teamIdx += 2;
                } else {
                    initialP2 = "BYE";
                    teamIdx += 1;
                }
            }

            batch.set(
                matchRef,
                {
                    id: match.id,
                    p1: initialP1,
                    p2: initialP2,
                    winner: null,
                    p1Score: null,
                    p2Score: null,
                    ...routingFields,
                },
                { merge: true },
            );
        }

        await batch.commit();
        alert("Success! Bracket generated and populated automatically.");
    }

    function recalculateVectors() {
        if (!containerEl) return;
        const containerRect = containerEl.getBoundingClientRect(),
            pathsTemp: { d: string; color: string }[] = [];

        const rightGapLanes: Record<string, number> = {},
            leftGapLanes: Record<string, number> = {};
        for (let route of bracketData.connections) {
            if (route.type === "loser" && !useColorLines) continue;
            const originNode = containerEl.querySelector(
                    `[data-match-id="${route.fromId}"]`,
                ),
                targetSlot = containerEl.querySelector(
                    `[data-match-id="${route.toId}"] [data-slot="${route.toSlot}"]`,
                );
            if (!originNode || !targetSlot) continue;

            const fromRect = originNode.getBoundingClientRect(),
                toSlotRect = targetSlot.getBoundingClientRect();
            const x1 =
                    route.type === "loser"
                        ? fromRect.left - containerRect.left
                        : fromRect.right - containerRect.left,
                y1 = fromRect.top + fromRect.height / 2 - containerRect.top;
            const x2 = toSlotRect.left - containerRect.left,
                y2 = toSlotRect.top + toSlotRect.height / 2 - containerRect.top;
            let color = "#1e293b";
            if (doubleElimEnabled && useColorLines)
                color = route.type === "winner" ? "#ef4444" : "#2563eb";

            let pathString = "";
            if (route.type === "loser") {
                const gapKey = `${Math.round(x1)}`;
                if (leftGapLanes[gapKey] === undefined)
                    leftGapLanes[gapKey] = 0;
                const lane = leftGapLanes[gapKey]++,
                    turnX1 = x1 - 25 - lane * 8;
                pathString = `M ${x1} ${y1} L ${turnX1} ${y1} L ${turnX1} ${y2} L ${x2} ${y2}`;
            } else {
                const gapKey = `${Math.round(x1)}-${Math.round(x2)}`;
                if (rightGapLanes[gapKey] === undefined)
                    rightGapLanes[gapKey] = 0;
                const lane = rightGapLanes[gapKey]++,
                    midX = x1 + (x2 - x1) * 0.4 + lane * 8;
                pathString = `M ${x1} ${y1} L ${midX} ${y1} L ${midX} ${y2} L ${x2} ${y2}`;
            }
            pathsTemp.push({ d: pathString, color });
        }
        svgPaths = pathsTemp;

        if (doubleElimEnabled) {
            const lastWB = bracketData.wb[bracketData.wb.length - 1]?.[0],
                lastLB = bracketData.lb[bracketData.lb.length - 1]?.[0];
            if (lastWB && lastLB) {
                const wbNode = containerEl.querySelector(
                        `[data-match-id="${lastWB.id}"]`,
                    ),
                    lbNode = containerEl.querySelector(
                        `[data-match-id="${lastLB.id}"]`,
                    );
                if (wbNode && lbNode) {
                    const wbRect = wbNode.getBoundingClientRect(),
                        lbRect = lbNode.getBoundingClientRect(),
                        centerPoint =
                            (wbRect.top +
                                wbRect.height / 2 +
                                (lbRect.top + lbRect.height / 2)) /
                            2;
                    championshipTop = centerPoint - containerRect.top - 50;
                }
            }
        }
    }

    $effect(() => {
        const _trigger = bracketData,
            _colorTrigger = useColorLines,
            _matchesTrigger = dbMatches,
            _isAdminTrigger = isAdmin;

        // Run immediately after next paint so DOM is settled
        let raf = requestAnimationFrame(() => {
            recalculateVectors();
            // Second pass to catch any remaining layout shifts
            raf = requestAnimationFrame(recalculateVectors);
        });

        let resizeObserver: ResizeObserver | null = null;
        let mutationObserver: MutationObserver | null = null;

        if (containerEl) {
            resizeObserver = new ResizeObserver(() => recalculateVectors());
            resizeObserver.observe(containerEl);

            // Watch for any DOM changes inside the bracket (e.g. admin selects expanding)
            mutationObserver = new MutationObserver(() => {
                requestAnimationFrame(recalculateVectors);
            });
            mutationObserver.observe(containerEl, {
                subtree: true,
                childList: true,
                attributes: true,
                attributeFilter: ["style", "class"],
            });
        }

        return () => {
            cancelAnimationFrame(raf);
            resizeObserver?.disconnect();
            mutationObserver?.disconnect();
        };
    });
</script>

{#if isAdmin}
    <div class="admin-dashboard-panel">
        <div class="admin-control-group">
            <label for="team-count-select">Bracket Size (team_count)</label>
            <input
                id="team-count-select"
                value={teamCount}
                onchange={(e) =>
                    handleUpdateTeamCount(Number(e.currentTarget.value))}
                class="admin-select"
                type="number"
                min="0"
            />
        </div>

        <div class="admin-control-group">
            <label for="double-elim-toggle">Format</label>
            <label class="admin-toggle-switch">
                <input
                    id="double-elim-toggle"
                    type="checkbox"
                    checked={doubleElimEnabled}
                    onchange={(e) =>
                        handleUpdateDoubleElimination(e.currentTarget.checked)}
                />
                <span class="admin-toggle-slider"></span>
                <span class="admin-toggle-caption">
                    {doubleElimEnabled
                        ? "Double Elimination"
                        : "Single Elimination"}
                </span>
            </label>
        </div>

        <div class="admin-control-group">
            <label>Stadiums</label>
            <button
                type="button"
                onclick={() => (showStadiumManager = !showStadiumManager)}
                class="admin-btn secondary-variant"
            >
                🏟️ {showStadiumManager ? "Hide" : "Manage"} Stadiums
            </button>
        </div>

        <div class="admin-control-group">
            <label for="new-team-input">Create & Add New Team</label>
            <div class="input-stack">
                <input
                    id="new-team-input"
                    type="text"
                    placeholder="Team name, e.g. Shadow Wolves"
                    bind:value={newTeamName}
                    class="admin-input"
                />
                <input
                    type="text"
                    placeholder="Logo URL (optional)"
                    bind:value={newTeamLogoUrl}
                    class="admin-input"
                />
                <input
                    type="text"
                    placeholder="Players, comma separated"
                    bind:value={newTeamPlayersInput}
                    class="admin-input"
                />
                <button type="button" onclick={handleAddTeam} class="admin-btn"
                    >Add Team</button
                >
            </div>
        </div>
    </div>

    {#if dbTeams.length > 0}
        <div class="team-roster-panel">
            <div class="team-roster-header">Teams ({dbTeams.length})</div>
            <div class="team-roster-grid">
                {#each dbTeams as team (team.id)}
                    <div class="team-roster-card">
                        {#if editingTeamId === team.id}
                            <div class="input-stack">
                                <input
                                    type="text"
                                    class="admin-input"
                                    bind:value={editTeamName}
                                    placeholder="Team name"
                                />
                                <input
                                    type="text"
                                    class="admin-input"
                                    bind:value={editTeamLogoUrl}
                                    placeholder="Logo URL"
                                />
                                <input
                                    type="text"
                                    class="admin-input"
                                    bind:value={editTeamPlayersInput}
                                    placeholder="Players, comma separated"
                                />
                                <div class="input-inline-row">
                                    <button
                                        type="button"
                                        class="admin-btn"
                                        onclick={handleSaveTeamEdits}
                                        >Save</button
                                    >
                                    <button
                                        type="button"
                                        class="admin-btn secondary-variant"
                                        onclick={cancelEditTeam}>Cancel</button
                                    >
                                </div>
                            </div>
                        {:else}
                            <div class="team-roster-card-head">
                                {#if team.logoUrl}
                                    <img
                                        src={team.logoUrl}
                                        alt=""
                                        class="team-logo-thumb"
                                    />
                                {/if}
                                <span class="team-roster-name">{team.name}</span
                                >
                            </div>
                            <div class="team-roster-players">
                                {team.players.length > 0
                                    ? team.players.join(", ")
                                    : "No players added"}
                            </div>
                            <div class="input-inline-row">
                                <button
                                    type="button"
                                    class="admin-btn secondary-variant"
                                    onclick={() => startEditTeam(team)}
                                    >Edit</button
                                >
                                <button
                                    type="button"
                                    class="admin-btn danger-variant"
                                    onclick={() => handleDeleteTeam(team.id)}
                                    >Delete</button
                                >
                            </div>
                        {/if}
                    </div>
                {/each}
            </div>
        </div>
    {/if}

    {#if showStadiumManager}
        <div class="team-roster-panel">
            <div class="team-roster-header">Stadiums & Livestreams</div>
            <div class="team-roster-grid">
                {#each dbStadiums as stadium (stadium.id)}
                    <div class="team-roster-card">
                        {#if editingStadiumId === stadium.id}
                            <div class="input-stack">
                                <input
                                    type="text"
                                    class="admin-input"
                                    bind:value={editStadiumName}
                                    placeholder="Stadium name"
                                />
                                <input
                                    type="url"
                                    class="admin-input"
                                    bind:value={editStadiumUrl}
                                    placeholder="Livestream URL (YouTube embed, etc.)"
                                />
                                <div class="input-inline-row">
                                    <button
                                        type="button"
                                        class="admin-btn"
                                        onclick={handleSaveStadiumEdits}
                                        >Save</button
                                    >
                                    <button
                                        type="button"
                                        class="admin-btn secondary-variant"
                                        onclick={() =>
                                            (editingStadiumId = null)}
                                        >Cancel</button
                                    >
                                </div>
                            </div>
                        {:else}
                            <div class="team-roster-card-head">
                                <span style="font-size: 1.1rem;">🏟️</span>
                                <span class="team-roster-name"
                                    >{stadium.name}</span
                                >
                            </div>
                            {#if stadium.livestreamUrl}
                                <div
                                    class="team-roster-players"
                                    style="color: #2563eb;"
                                >
                                    🔴 Stream configured
                                </div>
                            {:else}
                                <div class="team-roster-players">
                                    No stream URL
                                </div>
                            {/if}
                            <div class="input-inline-row">
                                <button
                                    type="button"
                                    class="admin-btn secondary-variant"
                                    onclick={() => startEditStadium(stadium)}
                                    >Edit</button
                                >
                                <button
                                    type="button"
                                    class="admin-btn danger-variant"
                                    onclick={() =>
                                        handleDeleteStadium(stadium.id)}
                                    >Delete</button
                                >
                            </div>
                        {/if}
                    </div>
                {/each}
                <div class="team-roster-card" style="border-style: dashed;">
                    <div
                        class="team-roster-name"
                        style="font-size: 0.8rem; color: #64748b;"
                    >
                        Add Stadium
                    </div>
                    <div class="input-stack">
                        <input
                            type="text"
                            class="admin-input"
                            bind:value={newStadiumName}
                            placeholder="Stadium / Field name"
                        />
                        <input
                            type="url"
                            class="admin-input"
                            bind:value={newStadiumUrl}
                            placeholder="Livestream embed URL (optional)"
                        />
                        <button
                            type="button"
                            class="admin-btn"
                            onclick={handleAddStadium}>Add Stadium</button
                        >
                    </div>
                </div>
            </div>
        </div>
    {/if}
{/if}

<div class="scrollable-viewport-container">
    <div
        class="fluid-workspace-canvas"
        style="padding-right: {doubleElimEnabled ? '24rem' : '6rem'};"
        bind:this={containerEl}
    >
        <svg class="circuit-vector-stage">
            {#each svgPaths as line}
                <path
                    d={line.d}
                    stroke={line.color}
                    stroke-width="2.5"
                    fill="none"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                />
            {/each}
        </svg>

        <div class="bracket-streams-grid">
            <div class="bracket-row-lane">
                <div class="lane-title style-w">Winners Bracket</div>
                <div class="columns-row" style="height: {laneHeights.wb}px;">
                    {#each bracketData.wb as round, rIdx}
                        <div class="tree-column">
                            <div class="round-label">
                                {rIdx === bracketData.wb.length - 1 &&
                                !doubleElimEnabled
                                    ? "Championship"
                                    : `WB Round ${rIdx + 1}`}
                            </div>
                            <div class="vertical-nodes-box">
                                {#each round as match}
                                    {@const dbM = dbMatches[match.id]}
                                    <div
                                        class="match-node-card {dbM?.isLive
                                            ? 'match-card-live'
                                            : ''}"
                                        style="position: absolute; top: {yPositions[
                                            match.id
                                        ] || 0}px;"
                                        data-match-id={match.id}
                                        onclick={() =>
                                            openMatchDetails(match.id)}
                                        role="button"
                                        tabindex="0"
                                    >
                                        {#if dbM?.stadium || dbM?.isLive || dbM?.startTime}
                                            <div class="match-status-strip">
                                                {#if dbM?.isLive}
                                                    <span class="live-pill"
                                                        >● LIVE</span
                                                    >
                                                {/if}
                                                {#if dbM?.stadium}
                                                    <span class="stadium-pill"
                                                        >{dbM.stadium}</span
                                                    >
                                                {/if}
                                                {#if !dbM?.isLive && dbM?.startTime}
                                                    {@const timeLabel =
                                                        getStartTimeLabel(dbM)}
                                                    {#if timeLabel}
                                                        <span
                                                            class="start-time-pill {getStartTimeLabel(
                                                                dbM,
                                                            )?.startsWith(
                                                                'Starting',
                                                            )
                                                                ? 'start-time-soon'
                                                                : ''}"
                                                            >{timeLabel}</span
                                                        >
                                                    {/if}
                                                {/if}
                                            </div>
                                        {/if}
                                        <div
                                            class="competitor-slot {match.p1 ===
                                            'BYE'
                                                ? 'bye-slot'
                                                : ''} {dbM?.winner &&
                                            dbM.winner === dbM.p1
                                                ? 'winner-highlight'
                                                : ''}"
                                            data-slot="p1"
                                        >
                                            {#if isAdmin}
                                                <div
                                                    class="slot-flex-row"
                                                    onclick={(e) =>
                                                        e.stopPropagation()}
                                                >
                                                    <select
                                                        value={dbM?.p1 || ""}
                                                        onchange={(e) =>
                                                            handleUpdateMatchSlot(
                                                                match.id,
                                                                "p1",
                                                                e.currentTarget
                                                                    .value,
                                                            )}
                                                        class="slot-embed-select"
                                                    >
                                                        <option value=""
                                                            >Default: {match.p1}</option
                                                        >
                                                        <option value="BYE"
                                                            >BYE</option
                                                        >
                                                        {#each dbTeams as team}
                                                            <option
                                                                value={team.id}
                                                                >{team.name}</option
                                                            >
                                                        {/each}
                                                    </select>
                                                    <span
                                                        class="match-score-badge"
                                                        >{dbM?.isLive
                                                            ? "?"
                                                            : (dbM?.p1Score ??
                                                              "-")}</span
                                                    >
                                                </div>
                                            {:else}
                                                <div class="user-slot-row">
                                                    <span class="team-identity">
                                                        {#if teamLogoFor(dbM?.p1)}
                                                            <img
                                                                src={teamLogoFor(
                                                                    dbM?.p1,
                                                                )}
                                                                alt=""
                                                                class="team-logo-inline"
                                                            />
                                                        {/if}
                                                        {match.p1}
                                                    </span>
                                                    <span
                                                        class="match-score-badge"
                                                        >{dbM?.isLive
                                                            ? "?"
                                                            : (dbM?.p1Score ??
                                                              "-")}</span
                                                    >
                                                </div>
                                            {/if}
                                        </div>

                                        <div
                                            class="competitor-slot {match.p2 ===
                                            'BYE'
                                                ? 'bye-slot'
                                                : ''} {dbM?.winner &&
                                            dbM.winner === dbM.p2
                                                ? 'winner-highlight'
                                                : ''}"
                                            data-slot="p2"
                                        >
                                            {#if isAdmin}
                                                <div
                                                    class="slot-flex-row"
                                                    onclick={(e) =>
                                                        e.stopPropagation()}
                                                >
                                                    <select
                                                        value={dbM?.p2 || ""}
                                                        onchange={(e) =>
                                                            handleUpdateMatchSlot(
                                                                match.id,
                                                                "p2",
                                                                e.currentTarget
                                                                    .value,
                                                            )}
                                                        class="slot-embed-select"
                                                    >
                                                        <option value=""
                                                            >Default: {match.p2}</option
                                                        >
                                                        <option value="BYE"
                                                            >BYE</option
                                                        >
                                                        {#each dbTeams as team}
                                                            <option
                                                                value={team.id}
                                                                >{team.name}</option
                                                            >
                                                        {/each}
                                                    </select>
                                                    <span
                                                        class="match-score-badge"
                                                        >{dbM?.isLive
                                                            ? "?"
                                                            : (dbM?.p2Score ??
                                                              "-")}</span
                                                    >
                                                </div>
                                            {:else}
                                                <div class="user-slot-row">
                                                    <span class="team-identity">
                                                        {#if teamLogoFor(dbM?.p2)}
                                                            <img
                                                                src={teamLogoFor(
                                                                    dbM?.p2,
                                                                )}
                                                                alt=""
                                                                class="team-logo-inline"
                                                            />
                                                        {/if}
                                                        {match.p2}
                                                    </span>
                                                    <span
                                                        class="match-score-badge"
                                                        >{dbM?.isLive
                                                            ? "?"
                                                            : (dbM?.p2Score ??
                                                              "-")}</span
                                                    >
                                                </div>
                                            {/if}
                                        </div>
                                    </div>
                                {/each}
                            </div>
                        </div>
                    {/each}
                </div>
            </div>

            {#if doubleElimEnabled && bracketData.lb.length > 0}
                <div class="bracket-row-lane">
                    <div class="lane-title style-l">Losers Bracket</div>
                    <div
                        class="columns-row"
                        style="height: {laneHeights.lb}px;"
                    >
                        {#each bracketData.lb as round, rIdx}
                            <div class="tree-column">
                                <div class="round-label">
                                    LB Round {rIdx + 1}
                                </div>
                                <div class="vertical-nodes-box">
                                    {#each round as match}
                                        {@const dbM = dbMatches[match.id]}
                                        <div
                                            class="match-node-card lb-variant {dbM?.isLive
                                                ? 'match-card-live'
                                                : ''}"
                                            style="position: absolute; top: {yPositions[
                                                match.id
                                            ] || 0}px;"
                                            data-match-id={match.id}
                                            onclick={() =>
                                                openMatchDetails(match.id)}
                                            role="button"
                                            tabindex="0"
                                        >
                                            {#if dbM?.stadium || dbM?.isLive || dbM?.startTime}
                                                <div class="match-status-strip">
                                                    {#if dbM?.isLive}
                                                        <span class="live-pill"
                                                            >● LIVE</span
                                                        >
                                                    {/if}
                                                    {#if dbM?.stadium}
                                                        <span
                                                            class="stadium-pill"
                                                            >{dbM.stadium}</span
                                                        >
                                                    {/if}
                                                    {#if !dbM?.isLive && dbM?.startTime}
                                                        {@const timeLabel =
                                                            getStartTimeLabel(
                                                                dbM,
                                                            )}
                                                        {#if timeLabel}
                                                            <span
                                                                class="start-time-pill {timeLabel.startsWith(
                                                                    'Starting',
                                                                )
                                                                    ? 'start-time-soon'
                                                                    : ''}"
                                                                >{timeLabel}</span
                                                            >
                                                        {/if}
                                                    {/if}
                                                </div>
                                            {/if}
                                            <div
                                                class="competitor-slot {match.p1 ===
                                                'BYE'
                                                    ? 'bye-slot'
                                                    : ''} {dbM?.winner &&
                                                dbM.winner === dbM.p1
                                                    ? 'winner-highlight'
                                                    : ''}"
                                                data-slot="p1"
                                            >
                                                {#if isAdmin}
                                                    <div
                                                        class="slot-flex-row"
                                                        onclick={(e) =>
                                                            e.stopPropagation()}
                                                    >
                                                        <select
                                                            value={dbM?.p1 ||
                                                                ""}
                                                            onchange={(e) =>
                                                                handleUpdateMatchSlot(
                                                                    match.id,
                                                                    "p1",
                                                                    e
                                                                        .currentTarget
                                                                        .value,
                                                                )}
                                                            class="slot-embed-select"
                                                        >
                                                            <option value=""
                                                                >Default: {match.p1}</option
                                                            >
                                                            <option value="BYE"
                                                                >BYE</option
                                                            >
                                                            {#each dbTeams as team}
                                                                <option
                                                                    value={team.id}
                                                                    >{team.name}</option
                                                                >
                                                            {/each}
                                                        </select>
                                                        <span
                                                            class="match-score-badge"
                                                            >{dbM?.isLive
                                                                ? "?"
                                                                : (dbM?.p1Score ??
                                                                  "-")}</span
                                                        >
                                                    </div>
                                                {:else}
                                                    <div class="user-slot-row">
                                                        <span
                                                            class="team-identity"
                                                        >
                                                            {#if teamLogoFor(dbM?.p1)}
                                                                <img
                                                                    src={teamLogoFor(
                                                                        dbM?.p1,
                                                                    )}
                                                                    alt=""
                                                                    class="team-logo-inline"
                                                                />
                                                            {/if}
                                                            {match.p1}
                                                        </span>
                                                        <span
                                                            class="match-score-badge"
                                                            >{dbM?.isLive
                                                                ? "?"
                                                                : (dbM?.p1Score ??
                                                                  "-")}</span
                                                        >
                                                    </div>
                                                {/if}
                                            </div>

                                            <div
                                                class="competitor-slot {match.p2 ===
                                                'BYE'
                                                    ? 'bye-slot'
                                                    : ''} {dbM?.winner &&
                                                dbM.winner === dbM.p2
                                                    ? 'winner-highlight'
                                                    : ''}"
                                                data-slot="p2"
                                            >
                                                {#if isAdmin}
                                                    <div
                                                        class="slot-flex-row"
                                                        onclick={(e) =>
                                                            e.stopPropagation()}
                                                    >
                                                        <select
                                                            value={dbM?.p2 ||
                                                                ""}
                                                            onchange={(e) =>
                                                                handleUpdateMatchSlot(
                                                                    match.id,
                                                                    "p2",
                                                                    e
                                                                        .currentTarget
                                                                        .value,
                                                                )}
                                                            class="slot-embed-select"
                                                        >
                                                            <option value=""
                                                                >Default: {match.p2}</option
                                                            >
                                                            <option value="BYE"
                                                                >BYE</option
                                                            >
                                                            {#each dbTeams as team}
                                                                <option
                                                                    value={team.id}
                                                                    >{team.name}</option
                                                                >
                                                            {/each}
                                                        </select>
                                                        <span
                                                            class="match-score-badge"
                                                            >{dbM?.isLive
                                                                ? "?"
                                                                : (dbM?.p2Score ??
                                                                  "-")}</span
                                                        >
                                                    </div>
                                                {:else}
                                                    <div class="user-slot-row">
                                                        <span
                                                            class="team-identity"
                                                        >
                                                            {#if teamLogoFor(dbM?.p2)}
                                                                <img
                                                                    src={teamLogoFor(
                                                                        dbM?.p2,
                                                                    )}
                                                                    alt=""
                                                                    class="team-logo-inline"
                                                                />
                                                            {/if}
                                                            {match.p2}
                                                        </span>
                                                        <span
                                                            class="match-score-badge"
                                                            >{dbM?.isLive
                                                                ? "?"
                                                                : (dbM?.p2Score ??
                                                                  "-")}</span
                                                        >
                                                    </div>
                                                {/if}
                                            </div>
                                        </div>
                                    {/each}
                                </div>
                            </div>
                        {/each}
                    </div>
                </div>
            {/if}
        </div>

        {#if doubleElimEnabled}
            {@const dbChamp = dbMatches["CHAMPIONSHIP"]}
            <div
                class="championship-showcase-dock"
                style="position: absolute; right: 4rem; top: {championshipTop}px; width: 220px;"
            >
                <div
                    class="round-label default-championship-text {useColorLines
                        ? 'chromatic-active-text'
                        : ''}"
                >
                    Grand Championship
                </div>
                <div
                    class="match-node-card default-championship-box {useColorLines
                        ? 'chromatic-active-box'
                        : ''} {dbChamp?.isLive ? 'match-card-live' : ''}"
                    data-match-id="CHAMPIONSHIP"
                    onclick={() => openMatchDetails("CHAMPIONSHIP")}
                    role="button"
                    tabindex="0"
                >
                    {#if dbChamp?.stadium || dbChamp?.isLive || dbChamp?.startTime}
                        <div class="match-status-strip">
                            {#if dbChamp?.isLive}
                                <span class="live-pill">● LIVE</span>
                            {/if}
                            {#if dbChamp?.stadium}
                                <span class="stadium-pill"
                                    >{dbChamp.stadium}</span
                                >
                            {/if}
                            {#if !dbChamp?.isLive && dbChamp?.startTime}
                                {@const timeLabel = getStartTimeLabel(dbChamp)}
                                {#if timeLabel}
                                    <span
                                        class="start-time-pill {timeLabel.startsWith(
                                            'Starting',
                                        )
                                            ? 'start-time-soon'
                                            : ''}">{timeLabel}</span
                                    >
                                {/if}
                            {/if}
                        </div>
                    {/if}
                    <div
                        class="competitor-slot field-neutral-w {useColorLines
                            ? 'chromatic-w'
                            : ''} {dbChamp?.winner &&
                        dbChamp.winner === dbChamp.p1
                            ? 'winner-highlight'
                            : ''}"
                        data-slot="p1"
                    >
                        {#if isAdmin}
                            <div
                                class="slot-flex-row"
                                onclick={(e) => e.stopPropagation()}
                            >
                                <select
                                    value={dbChamp?.p1 || ""}
                                    onchange={(e) =>
                                        handleUpdateMatchSlot(
                                            "CHAMPIONSHIP",
                                            "p1",
                                            e.currentTarget.value,
                                        )}
                                    class="slot-embed-select"
                                >
                                    <option value=""
                                        >Default: Winners Undefeated</option
                                    >
                                    {#each dbTeams as team}
                                        <option value={team.id}
                                            >{team.name}</option
                                        >
                                    {/each}
                                </select>
                                <span class="match-score-badge"
                                    >{dbChamp?.isLive
                                        ? "?"
                                        : (dbChamp?.p1Score ?? "-")}</span
                                >
                            </div>
                        {:else}
                            <div class="user-slot-row">
                                <span class="team-identity">
                                    {#if teamLogoFor(dbChamp?.p1)}
                                        <img
                                            src={teamLogoFor(dbChamp?.p1)}
                                            alt=""
                                            class="team-logo-inline"
                                        />
                                    {/if}
                                    {dbChamp?.p1
                                        ? dbTeams.find(
                                              (t) => t.id === dbChamp.p1,
                                          )?.name || "Winners Undefeated"
                                        : "Winners Undefeated"}
                                </span>
                                <span class="match-score-badge"
                                    >{dbChamp?.isLive
                                        ? "?"
                                        : (dbChamp?.p1Score ?? "-")}</span
                                >
                            </div>
                        {/if}
                    </div>
                    <div
                        class="competitor-slot field-neutral-l {useColorLines
                            ? 'chromatic-l'
                            : ''} {dbChamp?.winner &&
                        dbChamp.winner === dbChamp.p2
                            ? 'winner-highlight'
                            : ''}"
                        data-slot="p2"
                    >
                        {#if isAdmin}
                            <div
                                class="slot-flex-row"
                                onclick={(e) => e.stopPropagation()}
                            >
                                <select
                                    value={dbChamp?.p2 || ""}
                                    onchange={(e) =>
                                        handleUpdateMatchSlot(
                                            "CHAMPIONSHIP",
                                            "p2",
                                            e.currentTarget.value,
                                        )}
                                    class="slot-embed-select"
                                >
                                    <option value=""
                                        >Default: Losers Survivor</option
                                    >
                                    {#each dbTeams as team}
                                        <option value={team.id}
                                            >{team.name}</option
                                        >
                                    {/each}
                                </select>
                                <span class="match-score-badge"
                                    >{dbChamp?.isLive
                                        ? "?"
                                        : (dbChamp?.p2Score ?? "-")}</span
                                >
                            </div>
                        {:else}
                            <div class="user-slot-row">
                                <span class="team-identity">
                                    {#if teamLogoFor(dbChamp?.p2)}
                                        <img
                                            src={teamLogoFor(dbChamp?.p2)}
                                            alt=""
                                            class="team-logo-inline"
                                        />
                                    {/if}
                                    {dbChamp?.p2
                                        ? dbTeams.find(
                                              (t) => t.id === dbChamp.p2,
                                          )?.name || "Losers Survivor"
                                        : "Losers Survivor"}
                                </span>
                                <span class="match-score-badge"
                                    >{dbChamp?.isLive
                                        ? "?"
                                        : (dbChamp?.p2Score ?? "-")}</span
                                >
                            </div>
                        {/if}
                    </div>
                </div>
            </div>
        {/if}
    </div>
</div>

{#if activeDetailsMatchId}
    {@const detailsMatch = dbMatches[activeDetailsMatchId]}
    {@const p1Team = dbTeams.find((t) => t.id === detailsMatch?.p1)}
    {@const p2Team = dbTeams.find((t) => t.id === detailsMatch?.p2)}
    {@const p1Name = detailsMatch?.p1 === "BYE" ? "BYE" : p1Team?.name || "TBD"}
    {@const p2Name = detailsMatch?.p2 === "BYE" ? "BYE" : p2Team?.name || "TBD"}
    {@const matchLivestream = getLivestreamForMatch(detailsMatch)}
    <div
        class="match-modal-backdrop"
        onclick={closeMatchDetails}
        role="presentation"
    >
        <div
            class="match-modal-card"
            onclick={(e) => e.stopPropagation()}
            role="dialog"
            aria-modal="true"
        >
            <div class="match-modal-header">
                <div class="match-modal-teams">
                    <div class="match-modal-team-chip">
                        {#if p1Team?.logoUrl}
                            <img
                                src={p1Team.logoUrl}
                                alt=""
                                class="modal-team-logo"
                            />
                        {:else}
                            <span class="modal-team-logo-placeholder">?</span>
                        {/if}
                        <span class="modal-team-name">{p1Name}</span>
                    </div>
                    <span class="modal-vs-badge">VS</span>
                    <div class="match-modal-team-chip">
                        {#if p2Team?.logoUrl}
                            <img
                                src={p2Team.logoUrl}
                                alt=""
                                class="modal-team-logo"
                            />
                        {:else}
                            <span class="modal-team-logo-placeholder">?</span>
                        {/if}
                        <span class="modal-team-name">{p2Name}</span>
                    </div>
                </div>
                <button
                    type="button"
                    class="match-modal-close-btn"
                    onclick={closeMatchDetails}
                    aria-label="Close"
                >
                    ✕
                </button>
            </div>

            {#if matchLivestream && detailsMatch?.isLive && !isAdmin}
                <div class="match-modal-stream">
                    <iframe
                        src={matchLivestream}
                        title="Live Stream"
                        frameborder="0"
                        allow="autoplay; encrypted-media"
                        allowfullscreen
                        class="stream-iframe"
                    ></iframe>
                </div>
            {/if}

            {#if isAdmin}
                {@const activeMatch = dbMatches[activeDetailsMatchId!]}
                <div class="match-modal-body">
                    <div class="admin-control-group">
                        <span
                            style="font-size: 0.75rem; font-weight: 700; text-transform: uppercase; color: #64748b; letter-spacing: 0.05em;"
                            >Winner</span
                        >
                        <select
                            class="admin-input winner-select"
                            bind:value={detailsWinnerInput}
                            style="margin-top: 0.25rem;"
                        >
                            <option value="">— No winner yet —</option>
                            {#if activeMatch?.p1 && activeMatch.p1 !== "BYE"}
                                {@const t1 = dbTeams.find(
                                    (t) => t.id === activeMatch.p1,
                                )}
                                <option value={activeMatch.p1}
                                    >{t1?.name || activeMatch.p1} 👑</option
                                >
                            {/if}
                            {#if activeMatch?.p2 && activeMatch.p2 !== "BYE"}
                                {@const t2 = dbTeams.find(
                                    (t) => t.id === activeMatch.p2,
                                )}
                                <option value={activeMatch.p2}
                                    >{t2?.name || activeMatch.p2} 👑</option
                                >
                            {/if}
                        </select>
                    </div>

                    <div class="admin-control-group">
                        <span
                            style="font-size: 0.75rem; font-weight: 700; text-transform: uppercase; color: #64748b; letter-spacing: 0.05em;"
                            >Stadium / Field</span
                        >
                        {#if dbStadiums.length > 0}
                            <select
                                class="admin-input"
                                bind:value={detailsStadiumIdInput}
                                style="margin-top: 0.25rem;"
                            >
                                <option value="">— None / Custom —</option>
                                {#each dbStadiums as s}
                                    <option value={s.id}
                                        >{s.name}{s.livestreamUrl
                                            ? " 🔴"
                                            : ""}</option
                                    >
                                {/each}
                            </select>
                        {/if}
                        {#if !detailsStadiumIdInput}
                            <input
                                id="stadium-input"
                                type="text"
                                class="admin-input"
                                placeholder="e.g. Field 3 (or pick above)"
                                bind:value={detailsStadiumInput}
                                style="margin-top: 0.25rem;"
                            />
                        {/if}
                    </div>

                    <div class="admin-control-group">
                        <span
                            style="font-size: 0.75rem; font-weight: 700; text-transform: uppercase; color: #64748b; letter-spacing: 0.05em;"
                            >Start Time</span
                        >
                        <input
                            type="datetime-local"
                            class="admin-input"
                            bind:value={detailsStartTimeInput}
                            style="margin-top: 0.25rem;"
                        />
                    </div>

                    <div class="admin-control-group">
                        <span
                            style="font-size: 0.75rem; font-weight: 700; text-transform: uppercase; color: #64748b; letter-spacing: 0.05em;"
                            >Set Scores</span
                        >
                        <div
                            style="display: flex; gap: 0.5rem; align-items: center; margin-top: 0.25rem;"
                        >
                            <input
                                type="text"
                                class="admin-input score-input-box"
                                placeholder="P1 Score"
                                bind:value={detailsP1ScoreInput}
                            />
                            <span style="font-weight: 600; color: #64748b;"
                                >:</span
                            >
                            <input
                                type="text"
                                class="admin-input score-input-box"
                                placeholder="P2 Score"
                                bind:value={detailsP2ScoreInput}
                            />
                        </div>
                    </div>

                    <label class="match-modal-live-toggle">
                        <input
                            type="checkbox"
                            bind:checked={detailsIsLiveInput}
                        />
                        <span>Mark this match as live</span>
                    </label>
                </div>
                <div class="match-modal-footer">
                    <button
                        type="button"
                        class="admin-btn secondary-variant"
                        onclick={closeMatchDetails}
                    >
                        Cancel
                    </button>
                    <button
                        type="button"
                        class="admin-btn"
                        onclick={handleSaveMatchDetails}
                    >
                        Save
                    </button>
                </div>
            {:else}
                <div class="match-modal-body">
                    {#if detailsMatch?.startTime}
                        {@const timeLabel = getStartTimeLabel(detailsMatch)}
                        {#if timeLabel}
                            <p
                                class="match-modal-readonly-line {timeLabel.startsWith(
                                    'Starting',
                                )
                                    ? 'start-time-alert'
                                    : ''}"
                            >
                                🕐 <strong>{timeLabel}</strong>
                            </p>
                        {/if}
                    {/if}
                    <p class="match-modal-readonly-line">
                        <strong>Stadium:</strong>
                        {detailsMatch?.stadium || "To be announced"}
                    </p>
                    <p class="match-modal-readonly-line">
                        <strong>Status:</strong>
                        {detailsMatch?.isLive
                            ? "🔴 Live now"
                            : "Not currently live"}
                    </p>
                    <p class="match-modal-readonly-line">
                        <strong>Score:</strong>
                        <span style="font-weight: 700; margin-left: 0.25rem;">
                            {detailsMatch?.isLive
                                ? "? - ?"
                                : `${detailsMatch?.p1Score ?? "-"} - ${detailsMatch?.p2Score ?? "-"}`}
                        </span>
                    </p>
                    {#if matchLivestream && !detailsMatch?.isLive}
                        <p
                            class="match-modal-readonly-line"
                            style="color: #64748b; font-size: 0.8rem;"
                        >
                            Stream will appear here when the match goes live.
                        </p>
                    {/if}
                </div>
            {/if}
        </div>
    </div>
{/if}

<style>
    .admin-dashboard-panel {
        display: flex;
        flex-direction: row;
        flex-wrap: wrap;
        gap: 2rem;
        background: #ffffff;
        border: 1px dashed #cbd5e1;
        border-radius: 12px;
        padding: 1.5rem;
        margin-bottom: 2rem;
        align-items: flex-end;
    }
    .admin-control-group {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }
    .admin-control-group label {
        font-size: 0.75rem;
        font-weight: 700;
        text-transform: uppercase;
        color: #64748b;
        letter-spacing: 0.05em;
    }
    .admin-select,
    .admin-input {
        padding: 0.5rem 0.75rem;
        font-size: 0.85rem;
        border: 1px solid #cbd5e1;
        border-radius: 6px;
        background: #f8fafc;
        outline: none;
        min-width: 160px;
    }
    .score-input-box {
        min-width: 70px;
        max-width: 90px;
        text-align: center;
    }
    .input-inline-row {
        display: flex;
        flex-direction: row;
        gap: 0.5rem;
    }
    .admin-btn {
        background: #1e293b;
        color: #ffffff;
        border: none;
        padding: 0.5rem 1rem;
        font-size: 0.85rem;
        font-weight: 600;
        border-radius: 6px;
        cursor: pointer;
        transition: background 0.2s;
    }
    .admin-btn:hover {
        background: #0f172a;
    }
    .automated-gen-btn {
        background-color: #2563eb;
    }
    .automated-gen-btn:hover {
        background-color: #1d4ed8;
    }

    .slot-flex-row {
        display: flex;
        align-items: center;
        width: 100%;
        justify-content: space-between;
        gap: 0.5rem;
    }
    .user-slot-row {
        display: flex;
        align-items: center;
        justify-content: space-between;
        width: 100%;
    }
    .team-identity {
        display: flex;
        align-items: center;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .match-score-badge {
        font-weight: 700;
        background: #f1f5f9;
        padding: 0.15rem 0.4rem;
        border-radius: 4px;
        font-size: 0.75rem;
        color: #334155;
        min-width: 18px;
        text-align: center;
        margin-left: auto;
        margin-right: 0.25rem;
    }
    .slot-embed-select {
        width: 100%;
        background: transparent;
        border: none;
        outline: none;
        font-size: 0.8rem;
        font-weight: 500;
        color: #1e293b;
        cursor: pointer;
        text-overflow: ellipsis;
    }
    .action-winner-tick {
        background: none;
        border: none;
        cursor: pointer;
        font-size: 0.85rem;
        color: #10b981;
        padding: 0 0.2rem;
        transition: transform 0.1s ease;
    }
    .action-winner-tick:hover {
        transform: scale(1.2);
    }
    .winner-highlight {
        background-color: #ecfdf5 !important;
        color: #065f46 !important;
        font-weight: 700 !important;
    }

    .scrollable-viewport-container {
        width: 100%;
        overflow: auto;
        border: 1px solid #e2e8f0;
        border-radius: 12px;
        background: #fafafa;
    }
    .fluid-workspace-canvas {
        position: relative;
        display: flex;
        flex-direction: row;
        padding-top: 5rem;
        padding-bottom: 4rem;
        padding-left: 6rem;
        width: max-content;
    }
    .circuit-vector-stage {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
        z-index: 1;
    }
    .bracket-streams-grid {
        display: flex;
        flex-direction: column;
        gap: 5rem;
    }
    .bracket-row-lane {
        display: flex;
        flex-direction: column;
        gap: 1.2rem;
    }
    .columns-row {
        position: relative;
        display: flex;
        flex-direction: row;
        gap: 12rem;
    }
    .tree-column {
        position: relative;
        display: flex;
        flex-direction: column;
        width: 220px;
    }
    .round-label {
        font-size: 0.7rem;
        font-weight: 700;
        text-transform: uppercase;
        color: #94a3b8;
        letter-spacing: 0.06em;
        margin-bottom: 3rem;
        text-align: center;
    }
    .vertical-nodes-box {
        position: relative;
        width: 100%;
        height: 100%;
        overflow: visible;
    }
    .match-node-card {
        background: #ffffff;
        border: 1px solid #1e293b;
        border-radius: 4px;
        width: 100%;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
        z-index: 2;
        cursor: pointer;
        transition: box-shadow 0.15s ease;
        position: relative;
        overflow: visible;
    }
    .match-node-card.match-card-live {
        border-left: 3px solid #dc2626;
    }
    .championship-showcase-dock {
        width: 220px;
    }
    .championship-showcase-dock .match-node-card {
        width: 100%;
        box-sizing: border-box;
    }
    .winner-select {
        width: 100%;
        font-weight: 600;
    }
    .lb-variant {
        border-color: #475569;
    }
    .competitor-slot {
        padding: 0.65rem 0.85rem;
        font-size: 0.8rem;
        font-weight: 500;
        color: #1e293b;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .bye-slot {
        color: #94a3b8;
        font-style: italic;
        background: #f8fafc;
    }
    .competitor-slot:first-child {
        border-bottom: 1px solid #e2e8f0;
    }
    .lane-title {
        font-size: 0.75rem;
        font-weight: 800;
        text-transform: uppercase;
        letter-spacing: 0.04em;
        border-left: 3px solid #1e293b;
        padding-left: 0.5rem;
        color: #1e293b;
    }
    .default-championship-box {
        border: 2px solid #1e293b;
    }
    .default-championship-text {
        color: #1e293b;
    }
    .field-neutral-w {
        background: #f8fafc;
        color: #1e293b;
    }
    .field-neutral-l {
        background: #f1f5f9;
        color: #334155;
    }
    .chromatic-active-box {
        border-color: #2563eb !important;
    }
    .chromatic-active-text {
        color: #2563eb !important;
    }
    .chromatic-w {
        background: #f0fdf4 !important;
        color: #166534 !important;
        font-weight: 600;
    }
    .chromatic-l {
        background: #fff7ed !important;
        color: #9a3412 !important;
        font-weight: 600;
    }

    /* --- Live / Stadium / Time status strip — floats above the match card --- */
    .match-status-strip {
        position: absolute;
        bottom: 100%;
        left: -1px;
        right: -1px;
        display: flex;
        align-items: center;
        gap: 0.35rem;
        padding: 0.18rem 0.5rem;
        background: #fff8f8;
        border: 1px solid #fca5a5;
        border-bottom: none;
        border-radius: 4px 4px 0 0;
        flex-wrap: nowrap;
        white-space: nowrap;
        overflow: hidden;
        pointer-events: none;
        z-index: 3;
    }
    .live-pill {
        font-size: 0.65rem;
        font-weight: 700;
        color: #dc2626;
        text-transform: uppercase;
        letter-spacing: 0.04em;
        animation: livePulse 1.6s ease-in-out infinite;
    }
    @keyframes livePulse {
        0%,
        100% {
            opacity: 1;
        }
        50% {
            opacity: 0.4;
        }
    }
    .stadium-pill {
        font-size: 0.65rem;
        font-weight: 600;
        color: #475569;
        background: #e2e8f0;
        padding: 0.1rem 0.4rem;
        border-radius: 4px;
    }

    /* --- Inline team logos --- */
    .team-logo-inline {
        width: 16px;
        height: 16px;
        border-radius: 50%;
        object-fit: cover;
        vertical-align: middle;
        margin-right: 0.4rem;
    }
    .team-logo-thumb {
        width: 28px;
        height: 28px;
        border-radius: 50%;
        object-fit: cover;
        flex-shrink: 0;
    }

    /* --- Team roster manager panel --- */
    .team-roster-panel {
        background: #ffffff;
        border: 1px dashed #cbd5e1;
        border-radius: 12px;
        padding: 1.25rem 1.5rem;
        margin-bottom: 2rem;
    }
    .team-roster-header {
        font-size: 0.75rem;
        font-weight: 700;
        text-transform: uppercase;
        letter-spacing: 0.05em;
        color: #64748b;
        margin-bottom: 0.9rem;
    }
    .team-roster-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
        gap: 0.9rem;
    }
    .team-roster-card {
        border: 1px solid #e2e8f0;
        border-radius: 8px;
        padding: 0.85rem;
        display: flex;
        flex-direction: column;
        gap: 0.6rem;
        background: #fafafa;
    }
    .team-roster-card-head {
        display: flex;
        align-items: center;
        gap: 0.6rem;
    }
    .team-roster-name {
        font-size: 0.9rem;
        font-weight: 700;
        color: #0f172a;
    }
    .team-roster-players {
        font-size: 0.78rem;
        color: #64748b;
        line-height: 1.4;
    }
    .input-stack {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        min-width: 220px;
    }
    .admin-btn.secondary-variant {
        background: transparent;
        color: #334155;
        border: 1px solid #cbd5e1;
    }
    .admin-btn.secondary-variant:hover {
        background: #f1f5f9;
    }
    .admin-btn.danger-variant {
        background: #fef2f2;
        color: #b91c1c;
    }
    .admin-btn.danger-variant:hover {
        background: #fee2e2;
    }

    /* --- Match Details Modal --- */
    .match-modal-backdrop {
        position: fixed;
        inset: 0;
        background: rgba(15, 23, 42, 0.35);
        backdrop-filter: blur(3px);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 200;
        padding: 1rem;
        cursor: default;
    }
    .match-modal-card {
        background: #ffffff;
        border-radius: 12px;
        width: 100%;
        max-width: 420px;
        box-shadow:
            0 20px 25px -5px rgba(0, 0, 0, 0.1),
            0 10px 10px -5px rgba(0, 0, 0, 0.04);
        overflow: hidden;
        cursor: default;
    }
    .match-modal-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 1rem 1.1rem 0.9rem 1.1rem;
        border-bottom: 1px solid #f1f5f9;
        gap: 0.5rem;
    }
    .match-modal-teams {
        display: flex;
        align-items: center;
        gap: 0.6rem;
        flex: 1;
        min-width: 0;
        flex-wrap: wrap;
    }
    .match-modal-team-chip {
        display: flex;
        align-items: center;
        gap: 0.4rem;
        min-width: 0;
    }
    .modal-team-logo {
        width: 28px;
        height: 28px;
        border-radius: 50%;
        object-fit: cover;
        flex-shrink: 0;
        border: 1px solid #e2e8f0;
    }
    .modal-team-logo-placeholder {
        width: 28px;
        height: 28px;
        border-radius: 50%;
        background: #f1f5f9;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 0.7rem;
        color: #94a3b8;
        flex-shrink: 0;
        border: 1px solid #e2e8f0;
    }
    .modal-team-name {
        font-size: 0.9rem;
        font-weight: 700;
        color: #0f172a;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        max-width: 100px;
    }
    .modal-vs-badge {
        font-size: 0.65rem;
        font-weight: 800;
        color: #94a3b8;
        letter-spacing: 0.06em;
        flex-shrink: 0;
        background: #f1f5f9;
        padding: 0.15rem 0.35rem;
        border-radius: 4px;
    }
    .match-modal-close-btn {
        background: none;
        border: none;
        font-size: 0.9rem;
        color: #94a3b8;
        cursor: pointer;
        padding: 0.2rem;
        flex-shrink: 0;
    }
    .match-modal-close-btn:hover {
        color: #0f172a;
    }
    .match-modal-body {
        padding: 1.1rem 1.25rem;
        display: flex;
        flex-direction: column;
        gap: 0.9rem;
    }
    .match-modal-stream {
        width: 100%;
        background: #000;
    }
    .stream-iframe {
        width: 100%;
        aspect-ratio: 16 / 9;
        display: block;
        border: none;
    }
    .match-modal-live-toggle {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        font-size: 0.85rem;
        font-weight: 500;
        color: #334155;
        cursor: pointer;
    }
    .match-modal-readonly-line {
        margin: 0;
        font-size: 0.85rem;
        color: #334155;
    }
    .match-modal-readonly-line.start-time-alert {
        color: #b45309;
        background: #fffbeb;
        padding: 0.4rem 0.6rem;
        border-radius: 6px;
        border: 1px solid #fde68a;
    }
    .match-modal-footer {
        display: flex;
        justify-content: flex-end;
        gap: 0.6rem;
        padding: 0 1.25rem 1.25rem 1.25rem;
    }

    /* --- Start time pill on match cards --- */
    .start-time-pill {
        font-size: 0.65rem;
        font-weight: 600;
        color: #475569;
        background: #e2e8f0;
        padding: 0.1rem 0.4rem;
        border-radius: 4px;
    }
    .start-time-pill.start-time-soon {
        background: #fef3c7;
        color: #92400e;
        animation: livePulse 2s ease-in-out infinite;
    }

    /* --- Double Elimination toggle switch --- */
    .admin-toggle-switch {
        display: flex;
        align-items: center;
        gap: 0.55rem;
        cursor: pointer;
        user-select: none;
        height: 2.1rem;
    }
    .admin-toggle-switch input {
        display: none;
    }
    .admin-toggle-slider {
        position: relative;
        width: 34px;
        height: 18px;
        flex-shrink: 0;
        background-color: #cbd5e1;
        border-radius: 20px;
        transition: background-color 0.2s ease;
    }
    .admin-toggle-slider::before {
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
    .admin-toggle-switch input:checked + .admin-toggle-slider {
        background-color: #2563eb;
    }
    .admin-toggle-switch input:checked + .admin-toggle-slider::before {
        transform: translateX(16px);
    }
    /* --- Responsive: tablet & mobile --- */
    @media (max-width: 900px) {
        .admin-dashboard-panel {
            flex-direction: column;
            gap: 1.2rem;
        }
        .input-stack {
            min-width: 0;
            width: 100%;
        }
        .tree-column {
            width: 180px;
        }
        .columns-row {
            gap: 5rem;
        }
    }

    @media (max-width: 640px) {
        .admin-dashboard-panel {
            padding: 1rem;
            gap: 1rem;
        }
        .admin-select,
        .admin-input {
            min-width: 0;
            width: 100%;
        }
        .team-roster-grid {
            grid-template-columns: 1fr;
        }
        .tree-column {
            width: 155px;
        }
        .columns-row {
            gap: 3rem;
        }
        .match-modal-card {
            max-width: 100%;
            border-radius: 8px;
        }
        .match-modal-teams {
            gap: 0.4rem;
        }
        .modal-team-name {
            max-width: 72px;
            font-size: 0.82rem;
        }
        .modal-team-logo,
        .modal-team-logo-placeholder {
            width: 22px;
            height: 22px;
        }
        .match-modal-body {
            padding: 0.9rem 1rem;
        }
        .match-modal-footer {
            padding: 0 1rem 1rem;
        }
        .championship-showcase-dock {
            right: 1rem !important;
            width: 155px !important;
        }
    }
</style>
