# SIH
Railway transportaion optimization and time efficient movement

import matplotlib.pyplot as plt
import networkx as nx
import numpy as np   # not sure if I really need this yet, but keeping it just in case
from IPython.display import display, clear_output
import time


class RailSimulation:
    def __init__(self):
        # Keeping station layout simple for now
        self.stations = {
            'A': {'pos': (0, 0), 'name': 'Station A'},
            'B': {'pos': (4, 0), 'name': 'Station B'}, 
            'C': {'pos': (8, 0), 'name': 'Station C'}
        }
        
        # Tracks are basically one-way connections
        self.tracks = [
            ('A', 'B', 'A→B'),
            ('B', 'C', 'B→C'),
            ('B', 'A', 'B→A'),
            ('C', 'B', 'C→B')
        ]
        
        # None means track is free
        self.track_status = {t[2]: None for t in self.tracks}
        
        # Defining some trains manually (would be nice to load from a config later)
        self.trains = [
            {'id': 'Rajdhani Express', 'priority': 3, 'position': 'A', 'delay': 0, 
             'next_stop': 'B', 'progress': 0, 'color': 'red', 'speed': 0.15, 'route': ['A', 'B', 'C']},
            {'id': 'Shatabdi Express', 'priority': 3, 'position': 'A', 'delay': 0,
             'next_stop': 'B', 'progress': 0, 'color': 'blue', 'speed': 0.12, 'route': ['A', 'B', 'C']},
            {'id': 'Jan Shatabdi', 'priority': 2, 'position': 'A', 'delay': 0,
             'next_stop': 'B', 'progress': 0, 'color': 'green', 'speed': 0.10, 'route': ['A', 'B']},
            {'id': 'Local Passenger', 'priority': 1, 'position': 'A', 'delay': 0,
             'next_stop': 'B', 'progress': 0, 'color': 'orange', 'speed': 0.08, 'route': ['A', 'B']},
            {'id': 'Goods Freight', 'priority': 1, 'position': 'C', 'delay': 0,
             'next_stop': 'B', 'progress': 0, 'color': 'brown', 'speed': 0.06, 'route': ['C', 'B', 'A']}
        ]
        
        self.time_tick = 0
        self.event_log = []
    
    # Helper to build track name
    def track_name(self, frm, to):
        return f"{frm}→{to}"
    
    def update(self):
        self.time_tick += 1
        self.event_log = []  # reset every tick
        
        # --- Move trains already on tracks ---
        for t in self.trains:
            if t['progress'] > 0:   # means it's on a journey
                t['progress'] += t['speed']
                
                if t['progress'] >= 1.0:  # Train has arrived
                    trk = self.track_name(t['position'], t['next_stop'])
                    self.track_status[trk] = None   # free the track
                    
                    prev = t['position']
                    t['position'] = t['next_stop']
                    t['progress'] = 0
                    
                    # Work out where it should go next
                    try:
                        idx = t['route'].index(t['position'])
                    except ValueError:
                        idx = 0  # shouldn't happen but meh
                    
                    if idx < len(t['route']) - 1:
                        t['next_stop'] = t['route'][idx + 1]
                    else:
                        # hacky way of flipping route for round trip
                        t['route'] = list(reversed(t['route']))
                        t['next_stop'] = t['route'][1]
                    
                    self.event_log.append(f"✅ {t['id']} reached {t['position']}")
        
        # --- Assign tracks to waiting trains ---
        for stn in ['A', 'B', 'C']:
            waiting = [x for x in self.trains if x['position'] == stn and x['progress'] == 0]
            
            if waiting:
                # Priority first, then who’s been stuck the longest
                waiting.sort(key=lambda x: (x['priority'], x['delay']), reverse=True)
                
                for t in waiting:
                    trk = self.track_name(stn, t['next_stop'])
                    
                    if self.track_status.get(trk) is None:
                        self.track_status[trk] = t['id']
                        t['progress'] = 0.1  # kick off
                        self.event_log.append(f"🚆 {t['id']} leaving {stn} → {t['next_stop']}")
                        break   # only one dep per tick
    
        # --- Update delays ---
        for t in self.trains:
            if t['progress'] == 0:   # stuck waiting
                trk = self.track_name(t['position'], t['next_stop'])
                if self.track_status.get(trk) != t['id']:
                    t['delay'] += 1
    
    def draw_network(self):
        plt.figure(figsize=(10, 4))
        
        G = nx.DiGraph()
        
        for s, info in self.stations.items():
            G.add_node(s, pos=info['pos'], name=info['name'])
        
        for t in self.tracks:
            G.add_edge(t[0], t[1], label=t[2])
        
        pos = nx.get_node_attributes(G, 'pos')
        
        nx.draw_networkx_nodes(G, pos, node_size=1200, node_color='lightblue',
                               node_shape='s', edgecolors='black', linewidths=2)
        
        edge_labels = nx.get_edge_attributes(G, 'label')
        
        # Show each track depending on occupancy
        for e in G.edges():
            trk = self.track_name(e[0], e[1])
            occ = self.track_status.get(trk)
            
            col = 'red' if occ else 'green'
            style = 'solid' if occ else 'dashed'
            width = 3 if occ else 1.5
            
            nx.draw_networkx_edges(G, pos, edgelist=[e], width=width,
                                   edge_color=col, style=style, arrows=True,
                                   arrowsize=15, arrowstyle='->')
        
        nx.draw_networkx_edge_labels(G, pos, edge_labels=edge_labels, font_size=8)
        
        labels = {node: f"{node}\n{self.stations[node]['name']}" for node in G.nodes()}
        nx.draw_networkx_labels(G, pos, labels, font_size=9, font_weight='bold')
        
        # Place trains
        for t in self.trains:
            if t['progress'] == 0:  # waiting at a station
                st_pos = self.stations[t['position']]['pos']
                plt.plot(st_pos[0], st_pos[1], 's', markersize=12, 
                         color=t['color'], markeredgecolor='black', markeredgewidth=1)
            else:
                start = self.stations[t['position']]['pos']
                end = self.stations[t['next_stop']]['pos']
                px = start[0] + (end[0] - start[0]) * t['progress']
                py = start[1] + (end[1] - start[1]) * t['progress']
                plt.plot(px, py, 's', markersize=10, color=t['color'],
                         markeredgecolor='black', markeredgewidth=1)
        
        plt.title(f"🚆 Rail Network | Time: {self.time_tick}min | "
                  f"Active: {sum(1 for o in self.track_status.values() if o)}/4 | "
                  f"Delay: {sum(tr['delay'] for tr in self.trains)}min",
                  fontsize=10, fontweight='bold')
        
        plt.xlim(-1, 9)
        plt.ylim(-1, 1)
        plt.axis('off')
        plt.tight_layout()
        plt.show()


# --- Run simulation ---
sim = RailSimulation()

print("🚆 Starting RailSim Commander - Let's see how this goes")
print("=" * 70)

for step in range(25):
    clear_output(wait=True)
    sim.update()
    sim.draw_network()
    
    print(f"\n⏱ Simulation Time: {sim.time_tick} minutes")
    print("📊 Network Status:")
    
    moving = [t for t in sim.trains if t['progress'] > 0]
    delayed = [t for t in sim.trains if t['delay'] > 0]
    
    print(f"   • Moving trains: {len(moving)}")
    print(f"   • Delayed trains: {len(delayed)}")
    print(f"   • Total delay: {sum(t['delay'] for t in sim.trains)} minutes")
    print(f"   • Track utilization: {sum(1 for x in sim.track_status.values() if x)}/{len(sim.tracks)}")
    
    print("\n🚆 Train Positions:")
    for t in sim.trains:
        if t['progress'] == 0:
            status = f"🟦 WAITING at {t['position']}"
        else:
            status = f"🟢 MOVING {t['position']}→{t['next_stop']} ({int(t['progress']*100)}%)"
        
        if t['delay'] > 10:
            d_flag = "🔴"
        elif t['delay'] > 5:
            d_flag = "🟡"
        else:
            d_flag = "🟢"
        
        print(f"   {d_flag} {t['id']}: {status} | Delay: {t['delay']}min")
    
    if sim.event_log:
        print("\n📢 Recent Events:")
        for msg in sim.event_log[-3:]:
            print(f"   • {msg}")
    
    print("=" * 70)
    time.sleep(2)

print("\n🎯 Simulation Finished!")
print("📈 Final Report:")
print(f"   • Total time: {sim.time_tick} minutes")
print(f"   • Max single delay: {max(t['delay'] for t in sim.trains)} minutes")
print(f"   • Avg delay per train: {sum(t['delay'] for t in sim.trains)/len(sim.trains):.1f} minutes")
print(f"   • Efficiency: {max(0, 100 - (sum(t['delay'] for t in sim.trains)/len(sim.trains))):.1f}%")

print("\n🚆 Final Train Status:")
for t in sim.trains:
    print(f"   • {t['id']}: At {t['position']} | Total delay: {t['delay']}min | Priority: {'⭐' * t['priority']}")





    IMAGES:  (https://github.com/user-attachments/assets/69395a5f-4d3b-4367-870b-381c7800fc35)
    (https://github.com/user-attachments/assets/08bf667b-aa36-444c-ad67-0c05f6b97f20)


