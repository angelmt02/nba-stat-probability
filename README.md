# nba-stat-probability
from flask import Flask, request, render_template
import numpy as np

app = Flask(__name__)

# Example function to fetch player stats from a mock database
def fetch_player_stats(player_name):
    """
    Mock function to fetch player stats. Replace with actual API or database query.
    """
    mock_data = {
        "LeBron James": [
            {"points": 25, "rebounds": 10, "assists": 8, "steals": 2, "blocks": 1},
            {"points": 30, "rebounds": 12, "assists": 7, "steals": 1, "blocks": 1},
            {"points": 28, "rebounds": 9, "assists": 10, "steals": 3, "blocks": 2},
            {"points": 15, "rebounds": 6, "assists": 4, "steals": 0, "blocks": 1},
            {"points": 32, "rebounds": 11, "assists": 9, "steals": 2, "blocks": 3},
        ]
    }
    return mock_data.get(player_name, [])

def calculate_probability(stats, category, target_value):
    """
    Calculate the probability of achieving a specific value in a given statistical category.
    """
    data = [game[category] for game in stats if category in game]
    if not data:
        return 0
    unique, counts = np.unique(data, return_counts=True)
    total_games = len(data)
    probabilities = counts / total_games
    prob_dist = dict(zip(unique, probabilities))
    return prob_dist.get(target_value, 0)

def calculate_stat_line_probability(stats, target_stat_line):
    """
    Calculate the probability of achieving a specific stat line.
    """
    joint_probability = 1.0
    for category, target_value in target_stat_line.items():
        prob = calculate_probability(stats, category, target_value)
        joint_probability *= prob
    return joint_probability

@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        # Get form data
        player_name = request.form.get("player_name")
        target_points = int(request.form.get("points"))
        target_rebounds = int(request.form.get("rebounds"))
        target_assists = int(request.form.get("assists"))
        target_steals = int(request.form.get("steals"))
        target_blocks = int(request.form.get("blocks"))
        
        # Fetch player stats
        stats = fetch_player_stats(player_name)
        if not stats:
            return render_template("index.html", error=f"No stats found for {player_name}.")

        # Calculate probabilities
        target_stat_line = {
            "points": target_points,
            "rebounds": target_rebounds,
            "assists": target_assists,
            "steals": target_steals,
            "blocks": target_blocks,
        }
        probability = calculate_stat_line_probability(stats, target_stat_line)

        return render_template(
            "index.html",
            player_name=player_name,
            target_stat_line=target_stat_line,
            probability=f"{probability:.2%}"
        )

    return render_template("index.html")

if __name__ == "__main__":
    app.run(debug=True)
