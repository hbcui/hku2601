# Directed-grid MAPD pilot

The first benchmark uses a directed graph of rail cells, two OHTs, six adjacent machine docks, and persistent initial pickup-delivery tasks and route controls. An OHT advances at most one cell each tick. The learned model predicts both OHT move logits and next task-phase logits from the current state, route lookahead and dock positions. The graph projection rejects moves off the route, into occupied cells, and into conflicting cells. The transition is evaluated with a 60-tick free rollout and an always-move comparator.

The simplified label generator supplies supervised trajectories. It is not called during learned rollout. This benchmark isolates durable task/route influence and multi-agent occupancy from speed, acceleration, dense resource interactions and late events.

Current omissions: no routing control after tick zero, no later task arrival, no continuous position or speed, and no full-fab resource graph. A high score here does not establish full-AMHS validity.
