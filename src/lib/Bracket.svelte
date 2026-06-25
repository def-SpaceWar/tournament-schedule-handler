<script lang="ts">
    export type Sport = "soccer" | "basketball" | "volleyball";
    export type AgeGroup = "open" | "under10" | "under14" | "under18";
    export type Sex = "male" | "female" | "both";

    let {
        divisionLabel,
        sex,
        ageGroup,
        sport,
        totalTeams = 8,
        isDoubleElimination = true,
        useColorLines = false,
    }: {
        divisionLabel: string;
        sex: Sex;
        ageGroup: AgeGroup;
        sport: Sport;
        totalTeams?: number;
        isDoubleElimination?: boolean;
        useColorLines?: boolean;
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

    let containerEl = $state<HTMLElement | null>(null),
        svgPaths = $state<{ d: string; color: string }[]>([]),
        championshipTop = $state<number>(0);

    const NODE_HEIGHT = 68,
        MIN_VERTICAL_GAP = 28,
        TOTAL_NODE_SPACING = NODE_HEIGHT + MIN_VERTICAL_GAP;

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

        const teamsCount = totalTeams || 0;
        if (teamsCount < 2) return { wb: [], lb: [], connections };

        const nextPower = Math.pow(2, Math.ceil(Math.log2(teamsCount))),
            numMatchesR1 = nextPower / 2,
            numDuoMatches = teamsCount - numMatchesR1;

        let round1Matches: Match[] = [],
            nextWB: { name: string; sourceId: string }[] = [],
            round1Losers: { name: string; sourceId: string }[] = [],
            teamIdx = 0;

        for (let m = 0; m < numMatchesR1; m++) {
            let shortMatchId = `WB-R1-M${m + 1}`,
                p1 = "",
                p2 = "";

            if (m < numDuoMatches) {
                p1 = `Team ${teamIdx + 1}`;
                p2 = `Team ${teamIdx + 2}`;
                teamIdx += 2;
            } else {
                p1 = `Team ${teamIdx + 1}`;
                p2 = "BYE";
                teamIdx += 1;
            }

            round1Matches.push({ id: shortMatchId, p1, p2 });

            if (p2 === "BYE") {
                nextWB.push({ name: p1, sourceId: shortMatchId });
            } else {
                nextWB.push({
                    name: `Winner ${shortMatchId}`,
                    sourceId: shortMatchId,
                });
                round1Losers.push({
                    name: `Loser ${shortMatchId}`,
                    sourceId: shortMatchId,
                });
            }
        }
        wbRounds.push(round1Matches);
        wbLosersByRound.push(round1Losers);

        let currentWB = [...nextWB],
            rIdx = 2;

        while (currentWB.length > 1) {
            let roundMatches: Match[] = [],
                nextWBTemp: typeof currentWB = [],
                roundLosers: { name: string; sourceId: string }[] = [];

            for (let i = 0; i < currentWB.length; i += 2) {
                let shortMatchId = `WB-R${rIdx}-M${Math.floor(i / 2) + 1}`;

                roundMatches.push({
                    id: shortMatchId,
                    p1: currentWB[i].name,
                    p2: currentWB[i + 1].name,
                });

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

                nextWBTemp.push({
                    name: `Winner ${shortMatchId}`,
                    sourceId: shortMatchId,
                });
                roundLosers.push({
                    name: `Loser ${shortMatchId}`,
                    sourceId: shortMatchId,
                });
            }
            wbRounds.push(roundMatches);
            wbLosersByRound.push(roundLosers);
            currentWB = nextWBTemp;
            rIdx++;
        }

        if (isDoubleElimination && wbRounds.length > 0) {
            let lastWB = wbRounds[wbRounds.length - 1][0];
            if (lastWB) registerConnection(lastWB.id, "CHAMPIONSHIP", "p1");
        }

        if (isDoubleElimination && wbLosersByRound.length > 0) {
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

                        const p1 = p1Obj ? p1Obj.name : "BYE";
                        const p2 = p2Obj ? p2Obj.name : "BYE";

                        if (p1 === "BYE" && p2 === "BYE") continue;

                        roundMatches.push({ id: matchId, p1, p2 });

                        if (p1 !== "BYE" && p2 === "BYE")
                            nextSurvivors.push({ name: p1, sourceId: matchId });
                        else if (p1 === "BYE" && p2 !== "BYE")
                            nextSurvivors.push({ name: p2, sourceId: matchId });
                        else
                            nextSurvivors.push({
                                name: `Winner ${matchId}`,
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

                            const p1 = p1Obj ? p1Obj.name : "BYE";
                            const p2 = p2Obj ? p2Obj.name : "BYE";

                            if (p1 === "BYE" && p2 === "BYE") continue;

                            stageAMatches.push({ id: matchId, p1, p2 });

                            if (p1 !== "BYE" && p2 === "BYE")
                                stageANext.push({
                                    name: p1,
                                    sourceId: matchId,
                                });
                            else if (p1 === "BYE" && p2 !== "BYE")
                                stageANext.push({
                                    name: p2,
                                    sourceId: matchId,
                                });
                            else
                                stageANext.push({
                                    name: `Winner ${matchId}`,
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

                        const p1 = p1Obj ? p1Obj.name : "BYE";
                        const p2 = p2Obj ? p2Obj.name : "BYE";

                        if (p1 === "BYE" && p2 === "BYE") continue;

                        stageBMatches.push({ id: matchId, p1, p2 });

                        if (p1 !== "BYE" && p2 === "BYE")
                            stageBNext.push({ name: p1, sourceId: matchId });
                        else if (p1 === "BYE" && p2 !== "BYE")
                            stageBNext.push({ name: p2, sourceId: matchId });
                        else
                            stageBNext.push({
                                name: `Winner ${matchId}`,
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

        if (isDoubleElimination) {
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
                    let prevId = roundMatches[i - 1].id;
                    let currId = roundMatches[i].id;
                    if (pos[currId] < pos[prevId] + TOTAL_NODE_SPACING)
                        pos[currId] = pos[prevId] + TOTAL_NODE_SPACING;
                }
            }
        }

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
        return {
            wb: maxWB + NODE_HEIGHT + 20,
            lb: maxLB + NODE_HEIGHT + 20,
        };
    });

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
            if (isDoubleElimination && useColorLines)
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

        if (isDoubleElimination) {
            const lastWB = bracketData.wb[bracketData.wb.length - 1]?.[0],
                lastLB = bracketData.lb[bracketData.lb.length - 1]?.[0];
            if (lastWB && lastLB) {
                const wbNode = containerEl.querySelector(
                    `[data-match-id="${lastWB.id}"]`,
                );
                const lbNode = containerEl.querySelector(
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
            timer = setTimeout(recalculateVectors, 100);

        if (containerEl) {
            const observer = new ResizeObserver(() => recalculateVectors());
            observer.observe(containerEl);
            return () => {
                clearTimeout(timer);
                observer.disconnect();
            };
        }
        return () => clearTimeout(timer);
    });
</script>

<div class="scrollable-viewport-container">
    <div
        class="fluid-workspace-canvas"
        style="padding-right: {isDoubleElimination ? '24rem' : '6rem'};"
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
                                !isDoubleElimination
                                    ? "Championship"
                                    : `WB Round ${rIdx + 1}`}
                            </div>
                            <div class="vertical-nodes-box">
                                {#each round as match}
                                    <div
                                        class="match-node-card"
                                        style="position: absolute; top: {yPositions[
                                            match.id
                                        ] || 0}px;"
                                        data-match-id={match.id}
                                    >
                                        <div
                                            class="competitor-slot {match.p1 ===
                                            'BYE'
                                                ? 'bye-slot'
                                                : ''}"
                                            data-slot="p1"
                                        >
                                            {match.p1}
                                        </div>
                                        <div
                                            class="competitor-slot {match.p2 ===
                                            'BYE'
                                                ? 'bye-slot'
                                                : ''}"
                                            data-slot="p2"
                                        >
                                            {match.p2}
                                        </div>
                                    </div>
                                {/each}
                            </div>
                        </div>
                    {/each}
                </div>
            </div>

            {#if isDoubleElimination && bracketData.lb.length > 0}
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
                                        <div
                                            class="match-node-card lb-variant"
                                            style="position: absolute; top: {yPositions[
                                                match.id
                                            ] || 0}px;"
                                            data-match-id={match.id}
                                        >
                                            <div
                                                class="competitor-slot {match.p1 ===
                                                'BYE'
                                                    ? 'bye-slot'
                                                    : ''}"
                                                data-slot="p1"
                                            >
                                                {match.p1}
                                            </div>
                                            <div
                                                class="competitor-slot {match.p2 ===
                                                'BYE'
                                                    ? 'bye-slot'
                                                    : ''}"
                                                data-slot="p2"
                                            >
                                                {match.p2}
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

        {#if isDoubleElimination}
            <div
                class="championship-showcase-dock"
                style="position: absolute; right: 4rem; top: {championshipTop}px;"
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
                        : ''}"
                    data-match-id="CHAMPIONSHIP"
                >
                    <div
                        class="competitor-slot field-neutral-w {useColorLines
                            ? 'chromatic-w'
                            : ''}"
                        data-slot="p1"
                    >
                        Winners Undefeated
                    </div>
                    <div
                        class="competitor-slot field-neutral-l {useColorLines
                            ? 'chromatic-l'
                            : ''}"
                        data-slot="p2"
                    >
                        Losers Survivor
                    </div>
                </div>
            </div>
        {/if}
    </div>
</div>

<style>
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
        padding-top: 4rem;
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
        gap: 9rem;
    }

    .tree-column {
        position: relative;
        display: flex;
        flex-direction: column;
        width: 195px;
    }

    .round-label {
        font-size: 0.7rem;
        font-weight: 700;
        text-transform: uppercase;
        color: #94a3b8;
        letter-spacing: 0.06em;
        margin-bottom: 1rem;
        text-align: center;
    }

    .vertical-nodes-box {
        position: relative;
        width: 100%;
        height: 100%;
    }

    .match-node-card {
        background: #ffffff;
        border: 1px solid #1e293b;
        border-radius: 4px;
        width: 100%;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
        z-index: 2;
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
</style>
