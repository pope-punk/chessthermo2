# Thermodynamic Search: Unified Framework

## 1. Core Quantities

For a position with legal moves {m_1, ..., m_N}:

    Q_i  = value of move m_i = -F(position after m_i)
           (opponent's free energy, negated — good for us means bad for them)

    T    = temperature = max(stddev(Q_1..Q_N), T_FLOOR)
           (measures how "spread out" the move qualities are)

    Z    = partition function = SUM_i exp(Q_i / T)
           (total Boltzmann weight — measures how many good options exist)

    F    = free energy = T * ln(Z)
           (what the position is "worth" — accounts for BOTH best moves AND optionality)

    S    = entropy = -SUM_i p_i * ln(p_i)
           (how spread out the probability mass is across moves)

    p_i  = Boltzmann probability = exp(Q_i / T) / Z
           (how likely move m_i is under the ensemble)

Key identity: F = <Q> + T*S
(Free energy = average move quality + temperature * entropy)
(A position is good if it has good moves AND many of them)


## 2. Leaf Evaluation (depth 0)

At the bottom of the search tree, a position is scored by x-ray mobility:

    U = SUM(XRAY[piece, square]) for our pieces
      - SUM(XRAY[piece, square]) for their pieces

XRAY[piece, square] = number of squares that piece COULD reach
on an empty board from that square. This is the "ideal gas" energy —
each piece contributes independently.

U is the internal energy. It measures raw piece activity
without considering interactions or move availability.


## 3. Depth-1 thermoSearch (one side's free energy)

Given a position, generate all N legal moves. For each move m_i,
compute the change in U incrementally (no make/undo needed):

    delta_i = XRAY[piece][to] - XRAY[piece][from]
            + XRAY[captured][captured_sq]   (if capture)

    Q_i = baseU + delta_i

    where baseU = U_us - U_them (computed once from the full board)

Then compute T, Z, F from {Q_1, ..., Q_N} as in section 1.

This gives F_1: the free energy considering one side's moves.
Cost: one board scan + N table lookups. Very fast (~100us).


## 4. Multi-Depth thermoSearch (the unified search)

    thermoSearch(position, depth):

        if depth == 0:
            return leafEval(position)     ... section 2

        if depth == 1:
            return incremental F_1        ... section 3 (fast path)

        moves = legal_moves(position)
        N = len(moves)

        STEP 1: SCREENING
        Get quick estimates for all N moves using cached/shallow search:

            quickQ_i = -thermoSearch(after m_i, depth - 2)
                       (or depth-1 if no cache at depth-2)

            Uses iterative deepening cache: when computing depth D,
            results from depth D-1 and D-2 are already cached.

        STEP 2: BOLTZMANN PRUNING
        Estimate temperature from quick values:

            T_est = max(stddev(quickQ_1..N), T_FLOOR)
            threshold = max(quickQ) - 3 * T_est

        Classify moves into three tiers:

            EXACT tier:   quickQ_i >= threshold        (~10-15 moves)
                          These contribute meaningfully to Z.
                          Evaluate at full depth.

            SAMPLE tier:  quickQ_i < threshold          (~15-20 moves)
                          Boltzmann weight < 5% individually.
                          Monte Carlo sample 2-3 of these,
                          weighted by exp(quickQ_i / T_est).
                          If any sampled move's full Q jumps above
                          threshold, PROMOTE it to EXACT tier.

        STEP 3: FULL EVALUATION
        For each move in EXACT tier (+ promoted samples):

            Q_i = -thermoSearch(after m_i, depth - 1)

        For remaining moves (unpromoted SAMPLE tier + unevaluated):

            Q_i = quickQ_i    (use the screening estimate)

        STEP 4: PARTITION FUNCTION
        Compute T, Z, F, S, p_i from final {Q_1, ..., Q_N}
        as in section 1.

        Cache result keyed by (depth, position).
        Return F.


## 5. Move Selection (bestMove)

    bestMove(position, maxDepth):

        for depth = 1, 2, 3, ..., maxDepth:
            thermoSearch(position, depth)
            (results cached at each level, feeding the next)

            if time_limit_exceeded: break

        At the final depth, the root thermoSearch has:
            - Q_i for every move (full-depth values)
            - p_i for every move (Boltzmann probabilities)
            - T, S, F for the position

        Selected move = argmax(Q_i)
            (the move that minimizes opponent's free energy)
            (equivalent to minimax at T -> 0)

        The Q_i, p_i, T, S, F are passed directly to the dashboard.


## 6. Dashboard (window into the search)

The dashboard displays the ROOT-LEVEL thermodynamic state
from the most recent move decision. Nothing is recomputed.

    Temperature (T):    from the root thermoSearch
    Free Energy (F):    from the root thermoSearch
    Entropy (S):        from the root thermoSearch
    Heat Capacity (C):  (<Q^2> - <Q>^2) / T^2, from root data
    Move probabilities: p_i for each legal move
    Chemical potential:  computed from cached thermoSearch values
                         mu[piece] = F(with piece) - F(without piece)
                         Uses cached depth-1 results, no extra search needed.

All values are EXACTLY what the engine used to make its decision.


## 7. Value Flow Summary

    leafEval (depth 0)
        |
        v
    thermoSearch depth 1  (incremental, ~100us)
        |   cached by fast_pos_key
        v
    thermoSearch depth 2  (reuses depth-1 cache, Boltzmann prunes to ~12 moves)
        |   cached by (depth, fast_pos_key)
        v
    thermoSearch depth 3  (reuses depth-2 cache, Boltzmann prunes)
        |   cached by (depth, fast_pos_key)
        v
    ...continue to target depth...
        |
        v
    ROOT thermoSearch     (has Q_i, p_i, T, S, F for all moves)
        |                          |
        v                          v
    bestMove = argmax(Q_i)    dashboard display


## 8. Symmetry

Both sides are ALWAYS evaluated by the same thermoSearch function.
When computing Q_i = -thermoSearch(after our move, depth - 1),
the recursive call evaluates the OPPONENT's position using the
SAME function. The negation handles the perspective flip.

There is no separate "our eval" vs "their eval."
There is no separate "search" vs "eval."
There is one function: thermoSearch.


## 9. Efficiency

    Boltzmann pruning:   ~30 moves -> ~12 exact + 2-3 sampled = ~15 total
                         Effective branching factor ~15 (vs ~30 unpruned)

    Iterative deepening: depth D search reuses depth D-1 and D-2 caches
                         Screening step (step 1) is nearly free

    Incremental depth-1: No make/undo at leaves. XRAY delta per move.
                         ~100us cold, ~10us cached

    Cache keyed by:      depth + ':' + fast_pos_key()
                         (position + depth = unique computation)

    Estimated times:
        depth 2:  ~30ms
        depth 3:  ~300ms    (easy difficulty)
        depth 4:  ~3s       (medium difficulty)
        depth 5:  ~15-30s   (hard difficulty, or time-limited depth 5)


## 10. Monte Carlo Sampling Detail

For the SAMPLE tier (moves below the 3T threshold):

    Sampling weights:  w_i = exp(quickQ_i / T_est)
    Sampling probability:  p_sample_i = w_i / SUM(w_j for j in SAMPLE tier)

    Draw K=3 moves from this distribution (without replacement).
    Evaluate each at full depth: fullQ_i = -thermoSearch(after m_i, depth-1)

    If fullQ_i >= threshold:
        PROMOTE: move m_i joins the EXACT tier
        (it was underestimated by the quick screen — a tactical surprise)

    If fullQ_i < threshold:
        Keep fullQ_i as the move's Q value (more accurate than quickQ)

    Unpicked SAMPLE moves keep their quickQ_i values.

This catches "horizon effect" surprises at minimal cost (3 extra evals
per node). The sampling is biased toward the most promising tail moves,
so it's efficient exploration.
