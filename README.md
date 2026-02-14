# EmotionBridge
//app.py - backend


from flask import Flask, request, jsonify
from flask_cors import CORS
import re

app = Flask(__name__)
CORS(app)

def detect_emotion(message):
    """Simple keyword-based emotion detection"""
    message_lower = message.lower()
    
    # Anger keywords
    anger_words = ['angry', 'mad', 'furious', 'hate', 'stupid', 'idiot', 'damn', 'hell', 'pissed']
    # Frustration keywords
    frustration_words = ['frustrated', 'annoyed', 'ugh', 'seriously', 'fed up', 'sick of', 'tired of']
    # Sadness keywords
    sadness_words = ['sad', 'depressed', 'hurt', 'upset', 'disappointed', 'unhappy', 'heartbroken']
    # Positive keywords
    positive_words = ['happy', 'glad', 'excited', 'grateful', 'thankful', 'love', 'wonderful', 'great']
    
    for word in anger_words:
        if word in message_lower:
            return "Anger", "#ff5252"  # Red
    for word in frustration_words:
        if word in message_lower:
            return "Frustration", "#ff9800"  # Orange
    for word in sadness_words:
        if word in message_lower:
            return "Sadness", "#2196f3"  # Blue
    for word in positive_words:
        if word in message_lower:
            return "Positive", "#4caf50"  # Green
    
    return "Neutral", "#9e9e9e"  # Gray

def generate_bridged_message(original, emotion):
    """Generate empathetic response based on detected emotion"""
    bridged_responses = {
        "Anger": "I'm feeling really upset about this situation. Could we talk through it calmly? I want to understand your perspective too.",
        "Frustration": "This situation has been difficult for me. I'd appreciate if we could find a solution together.",
        "Sadness": "I'm feeling hurt by what happened. I value our connection and would like to work through this.",
        "Positive": "I'm really happy about this! Thanks for being part of it. 😊",
        "Neutral": "Thanks for sharing that with me. I appreciate our conversation."
    }
    
    return bridged_responses.get(emotion, "I'd like to talk about this. Can we discuss it?")

@app.route('/analyze', methods=['POST'])
def analyze():
    data = request.get_json()
    message = data.get('message', '')
    
    emotion, color = detect_emotion(message)
    bridged = generate_bridged_message(message, emotion)
    
    return jsonify({
        'emotion': emotion,
        'color': color,
        'bridged': bridged,
        'original': message
    })

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(debug=True, port=5000)


    
