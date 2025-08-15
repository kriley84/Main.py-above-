# Main.py-above-
AccaEdge
from fastapi import FastAPI, Query
import pandas as pd

app = FastAPI(title="Premier League Accumulator AI (Static + xG)")

# ---------- Static, realistic-looking PL fixtures (win/draw/lose %, form, injuries, xG) ----------
fixtures_data = [
    {"home": "Liverpool", "away": "Bournemouth",
     "home_win": 72, "draw": 18, "away_win": 10,
     "home_form": "W W W D W", "away_form": "L D W L L",
     "home_injuries": ["Mo Salah (hamstring)"], "away_injuries": ["Philip Billing (knee)"],
     "home_xg": 2.3, "away_xg": 1.1},

    {"home": "Manchester City", "away": "Newcastle",
     "home_win": 68, "draw": 20, "away_win": 12,
     "home_form": "W D W W W", "away_form": "W W L D W",
     "home_injuries": ["Kevin De Bruyne (ankle)"], "away_injuries": ["Callum Wilson (hamstring)"],
     "home_xg": 2.4, "away_xg": 1.6},

    {"home": "Arsenal", "away": "Chelsea",
     "home_win": 57, "draw": 26, "away_win": 17,
     "home_form": "W W L W D", "away_form": "D L W L W",
     "home_injuries": ["Gabriel Jesus (knee)"], "away_injuries": ["Reece James (thigh)"],
     "home_xg": 2.1, "away_xg": 1.5},

    {"home": "Tottenham", "away": "Brighton",
     "home_win": 62, "draw": 22, "away_win": 16,
     "home_form": "W L W W L", "away_form": "D W W L L",
     "home_injuries": ["James Maddison (ankle)"], "away_injuries": ["Kaoru Mitoma (ankle)"],
     "home_xg": 2.0, "away_xg": 1.6},

    {"home": "West Ham", "away": "Everton",
     "home_win": 48, "draw": 30, "away_win": 22,
     "home_form": "L W L D W", "away_form": "D L L W D",
     "home_injuries": ["Jarrod Bowen (groin)"], "away_injuries": ["Dominic Calvert-Lewin (ankle)"],
     "home_xg": 1.6, "away_xg": 1.4},

    {"home": "Aston Villa", "away": "Crystal Palace",
     "home_win": 59, "draw": 28, "away_win": 13,
     "home_form": "W D W W L", "away_form": "L L D W D",
     "home_injuries": ["Ollie Watkins (hamstring)"], "away_injuries": ["Eberechi Eze (groin)"],
     "home_xg": 1.9, "away_xg": 1.3},

    {"home": "Manchester United", "away": "Fulham",
     "home_win": 66, "draw": 21, "away_win": 13,
     "home_form": "W W D W L", "away_form": "L D W L W",
     "home_injuries": ["Lisandro Martinez (foot)"], "away_injuries": ["Willian (knee)"],
     "home_xg": 2.0, "away_xg": 1.3},

    {"home": "Brentford", "away": "Nottingham Forest",
     "home_win": 54, "draw": 29, "away_win": 17,
     "home_form": "L D W L W", "away_form": "W L L D W",
     "home_injuries": ["Ivan Toney (suspended)"], "away_injuries": ["Taiwo Awoniyi (groin)"],
     "home_xg": 1.7, "away_xg": 1.4},

    {"home": "Sheffield United", "away": "Luton Town",
     "home_win": 44, "draw": 33, "away_win": 23,
     "home_form": "L L D W L", "away_form": "D L W L D",
     "home_injuries": ["John Fleck (leg)"], "away_injuries": ["Carlton Morris (knee)"],
     "home_xg": 1.2, "away_xg": 1.1},

    {"home": "Burnley", "away": "Wolves",
     "home_win": 42, "draw": 34, "away_win": 24,
     "home_form": "W L L D L", "away_form": "W D W L D",
     "home_injuries": ["Lyle Foster (illness)"], "away_injuries": ["Pedro Neto (hamstring)"],
     "home_xg": 1.3, "away_xg": 1.4},
]

fixtures_df = pd.DataFrame(fixtures_data)

# ---------- Helpers ----------
def _badge(prob: int) -> str:
    """Return color badge text with % embedded."""
    if prob >= 60:
        return f"green({prob}%)"
    elif prob >= 40:
        return f"orange({prob}%)"
    return f"red({prob}%)"

def _form_points(form_str: str) -> int:
    """Convert 'W D L W ...' to points over last 5 (W=3,D=1,L=0)."""
    pts = 0
    for t in form_str.split():
        if t.upper() == "W":
            pts += 3
        elif t.upper() == "D":
            pts += 1
    return pts  # 0..15

def _xg_share(xg_for: float, xg_against: float) -> float:
    """Return xG share (0..1)."""
    total = max(xg_for + xg_against, 1e-6)
    return xg_for / total

def _leg_score(win_prob: int, xg_for: float, xg_against: float, form_str: str) -> float:
    """
    Composite score used for ranking recs.
    Weights: 60% win prob, 25% xG share, 15% recent form.
    All mapped to 0..100 scale.
    """
    win_component = win_prob  # already 0..100
    xg_component = _xg_share(xg_for, xg_against) * 100
    form_component = (_form_points(form_str) / 15) * 100
    return 0.60 * win_component + 0.25 * xg_component + 0.15 * form_component

# ---------- Endpoints ----------
@app.get("/fixtures")
def get_fixtures():
    """
    Return each fixture as two team entries (home & away),
    with win/draw/lose %, color badge (by win%), xG and recent form/injuries.
    """
    results = []
    for _, r in fixtures_df.iterrows():
        # Home view
        results.append({
            "fixture": f"{r['home']} vs {r['away']}",
            "team": r["home"],
            "win_probability": r["home_win"],
            "draw_probability": r["draw"],
            "lose_probability": r["away_win"],
            "rating_color": _badge(r["home_win"]),
            "xg_for": r["home_xg"],
            "xg_against": r["away_xg"],
            "form": r["home_form"],
            "injuries": r["home_injuries"],
            "role": "home"
        })
        # Away view
        results.append({
            "fixture": f"{r['home']} vs {r['away']}",
            "team": r["away"],
            "win_probability": r["away_win"],
            "draw_probability": r["draw"],
            "lose_probability": r["home_win"],
            "rating_color": _badge(r["away_win"]),
            "xg_for": r["away_xg"],
            "xg_against": r["home_xg"],
            "form": r["away_form"],
            "injuries": r["away_injuries"],
            "role": "away"
        })
    return results

@app.get("/recommendations")
def get_recommendations(n_legs: int = Query(4, ge=1, le=10)):
    """
    Pick top N legs using composite score = 60% win% + 25% xG share + 15% form points.
    Returns both sides of each chosen fixture for context.
    """
    # Rank by the HOME side composite score; include both sides in output
    ranked = []
    for _, r in fixtures_df.iterrows():
        score_home = _leg_score(r["home_win"], r["home_xg"], r["away_xg"], r["home_form"])
        ranked.append((score_home, r))

    ranked.sort(key=lambda x: x[0], reverse=True)
    top = [r for _, r in ranked[:n_legs]]

    results = []
    for r in top:
        # Home pick (drives the ranking)
        results.append({
            "pick_side": "home",
            "fixture": f"{r['home']} vs {r['away']}",
            "team": r["home"],
            "score": round(_leg_score(r["home_win"], r["home_xg"], r["away_xg"], r["home_form"]), 1),
            "win_probability": r["home_win"],
            "draw_probability": r["draw"],
            "lose_probability": r["away_win"],
            "rating_color": _badge(r["home_win"]),
            "xg_for": r["home_xg"],
            "xg_against": r["away_xg"],
            "form": r["home_form"],
            "injuries": r["home_injuries"]
        })
        # Away context (so users see both sides)
        results.append({
            "pick_side": "away_context",
            "fixture": f"{r['home']} vs {r['away']}",
            "team": r["away"],
            "score": round(_leg_score(r["away_win"], r["away_xg"], r["home_xg"], r["away_form"]), 1),
            "win_probability": r["away_win"],
            "draw_probability": r["draw"],
            "lose_probability": r["home_win"],
            "rating_color": _badge(r["away_win"]),
            "xg_for": r["away_xg"],
            "xg_against": r["home_xg"],
            "form": r["away_form"],
            "injuries": r["away_injuries"]
        })
    return results

@app.post("/simulate/injury")
def simulate_injury(team: str, win_delta: int = Query(-20), xg_delta: float = Query(-0.2)):
    """
    Simulate injury impact:
      - win% adjusted by win_delta (default -20)
      - xG adjusted by xg_delta (default -0.2)
    """
    global fixtures_df
    for idx, r in fixtures_df.iterrows():
        if r["home"] == team:
            fixtures_df.at[idx, "home_win"] = max(min(r["home_win"] + win_delta, 100), 0)
            fixtures_df.at[idx, "home_xg"] = max(r["home_xg"] + xg_delta, 0)
        if r["away"] == team:
            fixtures_df.at[idx, "away_win"] = max(min(r["away_win"] + win_delta, 100), 0)
            fixtures_df.at[idx, "away_xg"] = max(r["away_xg"] + xg_delta, 0)
    return {"message": f"Injury simulated for {team}", "win_delta": win_delta, "xg_delta": xg_delta}

@app.post("/simulate/odds_shift")
def simulate_odds(team: str, shift: int):
    """
    Adjust a team's win probability by `shift` (positive or negative).
    """
    global fixtures_df
    for idx, r in fixtures_df.iterrows():
        if r["home"] == team:
            fixtures_df.at[idx, "home_win"] = max(min(r["home_win"] + shift, 100), 0)
        if r["away"] == team:
            fixtures_df.at[idx, "away_win"] = max(min(r["away_win"] + shift, 100), 0)
    return {"message": f"Odds shifted for {team} by {shift}%"}

@app.post("/reset")
def reset_data():
    """
    Reset all fixture data back to the original snapshot.
    """
    global fixtures_df
    fixtures_df = pd.DataFrame(fixtures_data)
    return {"message": "Data reset to original"}
