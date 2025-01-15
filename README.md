from flask import Flask, request, render_template
from nba_api.stats.endpoints import playergamelog
from nba_api.stats.static import players
import numpy as np
import logging

# Initialize Flask app
app = Flask(__name__)

# Configure logging
logging.basicConfig(level=logging.INFO)

def fetch_player_stats(player_name, season, playoffs=False):
    """
    Fetch player stats using nba_api.
    
    Args:
        player_name (str): Name of the player (e.g., "LeBron James").
        season (str): NBA season in the format "2022-23".
        playoffs (bool): Whether to fetch playoff stats (default: False).
    
    Returns:
        list[dict]: A list of game stats as dictionaries.
    """
    try:
        # Find player ID
        player = players.find_players_by_full_name(player_name)
        if not player:
            raise ValueError(f"Player '{player_name}' not found.")
        
        player_id = player[0]['id']
        gamelog = playergamelog.PlayerGameLog(player_id=player_id, season=season, season_type_all_star="Playoffs" if playoffs else "Regular Season")
        games = gamelog.get_normalized_dict()['PlayerGameLog']
        
        # Extract relevant stats
        stats = []
        for game in games:
            stats.append({
                "points": game['PTS'],
                "rebounds": game['REB'],
                "assists": game['AST'],
                "steals": game['STL'],
                "blocks": game['BLK']
            })
        return stats
    
    except Exception as e:
        logging.error(f"Error fetching stats: {e}")
        return []

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
        try:
            # Get form data
            player_name = request.form.get("player_name")
            season = request.form.get("season")
            playoffs = request.form.get("playoffs") == "on"
            target_points = int(request.form.get("points"))
            target_rebounds = int(request.form.get("rebounds"))
            target_assists = int(request.form.get("assists"))
            target_steals = int(request.form.get("steals"))
            target_blocks = int(request.form.get("blocks"))
            
            # Fetch player stats
            stats = fetch_player_stats(player_name, season, playoffs)
            if not stats:
                return render_template("index.html", error=f"No stats found for {player_name} in {season}.")
            
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
                season=season,
                playoffs=playoffs,
                target_stat_line=target_stat_line,
                probability=f"{probability:.2%}"
            )
        
        except Exception as e:
            logging.error(f"Error: {e}")
            return render_template("index.html", error="An unexpected error occurred. Please try again.")
    
    return render_template("index.html")

if __name__ == "__main__":
    app.run(debug=True)
