from flask import Flask, request, render_template
from nba_api.stats.endpoints import playergamelog
from nba_api.stats.static import players
import numpy as np
import logging
from functools import lru_cache

# Initialize Flask app
app = Flask(__name__)

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

@lru_cache(maxsize=128)
def fetch_player_stats(player_name, season, playoffs=False):
    """
    Fetch player stats using nba_api with caching.
    
    Args:
        player_name (str): Player's name.
        season (str): NBA season (e.g., "2022-23").
        playoffs (bool): Fetch playoff stats if True.
    
    Returns:
        list[dict]: Player stats per game.
    """
    try:
        # Find player ID
        player = players.find_players_by_full_name(player_name)
        if not player:
            raise ValueError(f"Player '{player_name}' not found.")
        
        player_id = player[0]['id']
        season_type = "Playoffs" if playoffs else "Regular Season"
        gamelog = playergamelog.PlayerGameLog(player_id=player_id, season=season, season_type_all_star=season_type)
        games = gamelog.get_normalized_dict()['PlayerGameLog']
        
        stats = [
            {
                "points": game['PTS'],
                "rebounds": game['REB'],
                "assists": game['AST'],
                "steals": game['STL'],
                "blocks": game['BLK']
            }
            for game in games
        ]
        logging.info(f"Fetched {len(stats)} games for {player_name} ({season}, {season_type}).")
        return stats
    
    except Exception as e:
        logging.error(f"Error fetching stats for {player_name}: {e}")
        return []

def calculate_probability(stats, category, target_value):
    """
    Calculate the probability of achieving a target value in a category.
    """
    data = [game[category] for game in stats if category in game]
    if not data:
        return 0
    unique, counts = np.unique(data, return_counts=True)
    probabilities = counts / len(data)
    prob_dist = dict(zip(unique, probabilities))
    return prob_dist.get(target_value, 0)

def calculate_stat_line_probability(stats, target_stat_line):
    """
    Calculate the joint probability of achieving a stat line.
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
            # Collect form data
            player_name = request.form.get("player_name")
            season = request.form.get("season")
            playoffs = request.form.get("playoffs") == "on"
            target_stat_line = {
                "points": int(request.form.get("points")),
                "rebounds": int(request.form.get("rebounds")),
                "assists": int(request.form.get("assists")),
                "steals": int(request.form.get("steals")),
                "blocks": int(request.form.get("blocks")),
            }

            # Fetch stats
            stats = fetch_player_stats(player_name, season, playoffs)
            if not stats:
                return render_template("index.html", error=f"No stats found for {player_name} in {season}.")
            
            # Calculate probability
            probability = calculate_stat_line_probability(stats, target_stat_line)
            return render_template(
                "index.html",
                player_name=player_name,
                season=season,
                playoffs=playoffs,
                target_stat_line=target_stat_line,
                probability=f"{probability:.2%}"
            )
        
        except ValueError as e:
            logging.error(f"Value Error: {e}")
            return render_template("index.html", error="Invalid input. Please try again.")
        except Exception as e:
            logging.error(f"Unexpected Error: {e}")
            return render_template("index.html", error="An unexpected error occurred. Please try again.")
    
    return render_template("index.html")

if __name__ == "__main__":
    app.run(debug=True)
