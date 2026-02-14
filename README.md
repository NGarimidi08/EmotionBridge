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


    
//App.js - frontend
import React, { useState, useEffect } from 'react';
import './App.css';

function App() {
  const [inputText, setInputText] = useState("");
  const [messages, setMessages] = useState([]);
  const [loading, setLoading] = useState(false);
  const [activeTab, setActiveTab] = useState('chat');
  const [privacyMode, setPrivacyMode] = useState(false);
  const [coolDown, setCoolDown] = useState({
    active: false,
    seconds: 30,
    message: '',
    emotion: '',
    bridged: '',
    color: ''
  });
  const [editingMessage, setEditingMessage] = useState(null);

  // Emotion color mapping
  const emotionColors = {
    Anger: "#FF5252",
    Frustration: "#F59E0B",
    Sadness: "#3B82F6",
    Positive: "#34D399",
    Neutral: "#9CA3AF"
  };

  // Cool down timer effect
  useEffect(() => {
    let timer;
    if (coolDown.active && coolDown.seconds > 0) {
      timer = setTimeout(() => {
        setCoolDown(prev => ({
          ...prev,
          seconds: prev.seconds - 1
        }));
      }, 1000);
    } else if (coolDown.seconds === 0) {
      handleSendBridgedMessage();
    }
    return () => clearTimeout(timer);
  }, [coolDown.active, coolDown.seconds]);

  const detectHighEmotion = (emotion) => {
    return emotion === 'Anger' || emotion === 'Frustration';
  };

  const handleAnalyze = async (e) => {
    e.preventDefault();
    if (!inputText.trim()) return;

    setLoading(true);
    try {
      const response = await fetch('http://localhost:5000/analyze', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ message: inputText }),
      });

      const data = await response.json();
      
      // Check if message is highly emotional and cool down should activate
      if (detectHighEmotion(data.emotion) && !privacyMode) {
        setCoolDown({
          active: true,
          seconds: 30,
          message: inputText,
          emotion: data.emotion,
          bridged: data.bridged,
          color: data.color || emotionColors[data.emotion]
        });
        setInputText('');
        setLoading(false);
        return;
      }

      // Add message normally
      addNewMessage(inputText, data);
      setInputText('');
    } catch (error) {
      console.error('Error:', error);
      alert('Failed to analyze message. Make sure the backend is running.');
    } finally {
      setLoading(false);
    }
  };

  const addNewMessage = (original, data, isRewind = false, originalId = null) => {
    const newMessage = {
      id: Date.now(),
      original: original,
      emotion: data.emotion,
      color: data.color || emotionColors[data.emotion],
      bridged: data.bridged,
      timestamp: new Date().toLocaleTimeString(),
      date: new Date().toISOString(),
      private: privacyMode,
      canRewind: !privacyMode,
      originalId: originalId,
      rewound: false
    };

    if (privacyMode) {
      // Show brief notification instead of saving
      alert(`🔒 Private Mode: "${data.bridged}"`);
    } else {
      setMessages(prev => [newMessage, ...prev]);
    }
  };

  const handleSendBridgedMessage = () => {
    if (coolDown.message) {
      const data = {
        emotion: coolDown.emotion,
        color: coolDown.color,
        bridged: coolDown.bridged
      };
      addNewMessage(coolDown.message, data);
      setCoolDown({ active: false, seconds: 30, message: '', emotion: '', bridged: '', color: '' });
    }
  };

  const cancelCoolDown = () => {
    setCoolDown({ active: false, seconds: 30, message: '', emotion: '', bridged: '', color: '' });
    setInputText(coolDown.message);
  };

  const handleRewind = (message) => {
    setEditingMessage(message);
  };

  const submitRewind = async (rewrittenMessage) => {
    if (!rewrittenMessage.trim() || !editingMessage) return;

    try {
      const response = await fetch('http://localhost:5000/analyze', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ message: rewrittenMessage }),
      });

      const data = await response.json();
      
      // Add as new message with reference to original
      addNewMessage(rewrittenMessage, data, true, editingMessage.id);
      
      // Mark original as rewound
      setMessages(prev => prev.map(msg => 
        msg.id === editingMessage.id 
          ? { ...msg, rewound: true, rewoundVersion: rewrittenMessage }
          : msg
      ));

      setEditingMessage(null);
    } catch (error) {
      console.error('Error:', error);
    }
  };

  // Analysis Functions
  const getEmotionStats = () => {
    const stats = {
      Anger: 0,
      Frustration: 0,
      Sadness: 0,
      Positive: 0,
      Neutral: 0,
      total: 0
    };

    messages.forEach(msg => {
      if (msg.emotion && !msg.private) {
        stats[msg.emotion]++;
        stats.total++;
      }
    });

    return stats;
  };

  const getMostUsedEmotions = () => {
    const stats = getEmotionStats();
    const emotions = Object.entries(stats)
      .filter(([key]) => key !== 'total')
      .sort((a, b) => b[1] - a[1])
      .slice(0, 3);

    return emotions;
  };

  const getTriggerWords = () => {
    const triggerWords = {
      anger: ['hate', 'stupid', 'idiot', 'damn', 'hell', 'angry', 'mad', 'furious'],
      frustration: ['frustrated', 'annoyed', 'ugh', 'seriously', 'fed up', 'sick of'],
      sadness: ['sad', 'lonely', 'hurt', 'disappointed', 'unhappy', 'depressed']
    };

    const wordCount = {};
    
    messages.forEach(msg => {
      if (!msg.private) {
        const words = msg.original.toLowerCase().split(/\s+/);
        words.forEach(word => {
          // Check if word is a trigger word
          for (const [emotion, triggers] of Object.entries(triggerWords)) {
            if (triggers.includes(word)) {
              wordCount[word] = (wordCount[word] || 0) + 1;
            }
          }
        });
      }
    });

    // Sort and get top 5
    return Object.entries(wordCount)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5);
  };

  const getEmpathyGrowth = () => {
    // Group messages by day and calculate emotion distribution
    const messagesByDay = {};
    
    messages.forEach(msg => {
      if (!msg.private && msg.date) {
        const day = new Date(msg.date).toLocaleDateString();
        if (!messagesByDay[day]) {
          messagesByDay[day] = { total: 0, positive: 0, negative: 0 };
        }
        messagesByDay[day].total++;
        if (msg.emotion === 'Positive') {
          messagesByDay[day].positive++;
        } else if (['Anger', 'Frustration', 'Sadness'].includes(msg.emotion)) {
          messagesByDay[day].negative++;
        }
      }
    });

    // Calculate empathy score for each day (positive/total ratio)
    const growth = Object.entries(messagesByDay)
      .map(([day, data]) => ({
        day: day.slice(0, 5), // Short format
        score: data.total > 0 ? Math.round((data.positive / data.total) * 100) : 0
      }))
      .slice(-7); // Last 7 days

    return growth;
  };

  const getImprovementTrend = () => {
    const growth = getEmpathyGrowth();
    if (growth.length < 2) return 0;
    
    const first = growth[0]?.score || 0;
    const last = growth[growth.length - 1]?.score || 0;
    return last - first;
  };

  const emotionStats = getEmotionStats();
  const mostUsedEmotions = getMostUsedEmotions();
  const triggerWords = getTriggerWords();
  const empathyGrowth = getEmpathyGrowth();
  const improvementTrend = getImprovementTrend();

  return (
    <div className="App">
      {/* Header */}
      <div className="header">
        <h1>EmotionBridge</h1>
        <p className="tagline">Say What You Feel, Not What You Regret</p>
        <button 
          className={`privacy-toggle ${privacyMode ? 'active' : ''}`}
          onClick={() => setPrivacyMode(!privacyMode)}
        >
          {privacyMode ? '🔒 Private Mode' : '🌐 Public Mode'}
        </button>
      </div>

      {/* Cool Down Modal */}
      {coolDown.active && (
        <div className="modal-overlay">
          <div className="modal-content">
            <div className="cool-down-icon">😤</div>
            <h2>Take a moment...</h2>
            <p className="cool-down-message">"{coolDown.message}"</p>
            <div className="timer">{coolDown.seconds}s</div>
            <p className="suggestion-text">Try this instead:</p>
            <p className="bridged-suggestion">"{coolDown.bridged}"</p>
            <div className="modal-actions">
              <button onClick={handleSendBridgedMessage} className="primary-btn">
                Send Calmer Version
              </button>
              <button onClick={cancelCoolDown} className="secondary-btn">
                Edit Original
              </button>
            </div>
          </div>
        </div>
      )}

      {/* Rewind Modal */}
      {editingMessage && (
        <div className="modal-overlay">
          <div className="modal-content">
            <h2>↻ Rewind & Rewrite</h2>
            <div className="original-message-block">
              <small>Original:</small>
              <p>"{editingMessage.original}"</p>
              <span className="emotion-badge" style={{ backgroundColor: editingMessage.color }}>
                {editingMessage.emotion}
              </span>
            </div>
            
            <textarea
              className="rewind-input"
              placeholder="Rewrite your message here..."
              defaultValue={editingMessage.original}
              rows="3"
            />

            <div className="suggestion-chips">
              <button 
                className="chip"
                onClick={() => submitRewind(editingMessage.bridged)}
              >
                Use suggested version
              </button>
              <button 
                className="chip"
                onClick={() => submitRewind(`I feel ${editingMessage.emotion.toLowerCase()} about this. Can we talk?`)}
              >
                Make it neutral
              </button>
            </div>

            <div className="modal-actions">
              <button 
                className="primary-btn"
                onClick={(e) => {
                  const textarea = document.querySelector('.rewind-input');
                  submitRewind(textarea.value);
                }}
              >
                Submit Rewind
              </button>
              <button 
                className="secondary-btn"
                onClick={() => setEditingMessage(null)}
              >
                Cancel
              </button>
            </div>
          </div>
        </div>
      )}

      {/* Main Input */}
      <div className="input-section">
        <input
          type="text"
          placeholder="Type your message here..."
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          disabled={loading || coolDown.active}
        />
        <button onClick={handleAnalyze} disabled={loading || !inputText.trim() || coolDown.active}>
          {loading ? '✨' : 'Bridge My Message'}
        </button>
      </div>

      {/* Tabs */}
      <div className="tabs">
        <button 
          className={`tab ${activeTab === 'chat' ? 'active' : ''}`}
          onClick={() => setActiveTab('chat')}
        >
          Chat
        </button>
        <button 
          className={`tab ${activeTab === 'suggestions' ? 'active' : ''}`}
          onClick={() => setActiveTab('suggestions')}
        >
          Suggestions
        </button>
        <button 
          className={`tab ${activeTab === 'rewind' ? 'active' : ''}`}
          onClick={() => setActiveTab('rewind')}
        >
          Rewind
        </button>
        <button 
          className={`tab ${activeTab === 'analysis' ? 'active' : ''}`}
          onClick={() => setActiveTab('analysis')}
        >
          Analysis
        </button>
      </div>

      {/* Content Area */}
      <div className="content-area">
        {activeTab === 'chat' && (
          <div className="messages-list">
            {messages.length === 0 ? (
              <p className="empty-state">
                No messages yet. Start a conversation!
              </p>
            ) : (
              messages.map((msg) => (
                <div key={msg.id} className="message-card">
                  <div className="message-header">
                    <span className="sender">You</span>
                    <span className="timestamp">{msg.timestamp}</span>
                  </div>
                  <p className="message-text">{msg.original}</p>
                  <div className="message-footer">
                    <span className="emotion-badge" style={{ backgroundColor: msg.color }}>
                      {msg.emotion}
                    </span>
                    {!msg.rewound && !privacyMode && (
                      <button 
                        className="rewind-btn"
                        onClick={() => handleRewind(msg)}
                      >
                        ↻ Rewind
                      </button>
                    )}
                  </div>
                </div>
              ))
            )}
          </div>
        )}

        {activeTab === 'suggestions' && (
          <div className="suggestions-list">
            {messages.filter(m => !m.private).length === 0 ? (
              <p className="empty-state">Send some messages to get suggestions</p>
            ) : (
              messages.filter(m => !m.private).map((msg) => (
                <div key={msg.id} className="suggestion-card">
                  <div className="suggestion-original">
                    <small>You said:</small>
                    <p>"{msg.original}"</p>
                    <span className="emotion-badge" style={{ backgroundColor: msg.color }}>
                      {msg.emotion}
                    </span>
                  </div>
                  <div className="suggestion-bridged">
                    <small>Try this:</small>
                    <p className="bridged-text">"{msg.bridged}"</p>
                  </div>
                  <button 
                    className="use-suggestion-btn"
                    onClick={() => {
                      setInputText(msg.bridged);
                      setActiveTab('chat');
                    }}
                  >
                    Use this response
                  </button>
                </div>
              ))
            )}
          </div>
        )}

        {activeTab === 'rewind' && (
          <div className="rewind-list">
            {messages.filter(m => m.rewound).length === 0 ? (
              <p className="empty-state">No rewritten messages yet. Click Rewind on any message to improve it.</p>
            ) : (
              messages.filter(m => m.rewound).map((msg) => (
                <div key={msg.id} className="rewind-card">
                  <div className="rewind-comparison">
                    <div className="original-version">
                      <small>Original</small>
                      <p>"{msg.original}"</p>
                    </div>
                    <span className="rewind-arrow">→</span>
                    <div className="rewound-version">
                      <small>Rewritten</small>
                      <p>"{msg.rewoundVersion}"</p>
                    </div>
                  </div>
                  <div className="rewind-footer">
                    <span className="timestamp">{msg.timestamp}</span>
                    <span className="emotion-badge" style={{ backgroundColor: msg.color }}>
                      {msg.emotion}
                    </span>
                  </div>
                </div>
              ))
            )}
          </div>
        )}

        {activeTab === 'analysis' && (
          <div className="analysis-dashboard">
            {/* Most Used Emotions */}
            <div className="analysis-section">
              <h3>Most Used Emotions</h3>
              <div className="emotion-bars">
                {mostUsedEmotions.length > 0 ? (
                  mostUsedEmotions.map(([emotion, count]) => (
                    <div key={emotion} className="emotion-bar-item">
                      <div className="emotion-bar-label">
                        <span className="emotion-dot" style={{ backgroundColor: emotionColors[emotion] }}></span>
                        <span>{emotion}</span>
                        <span className="emotion-count">{count}</span>
                      </div>
                      <div className="emotion-bar-container">
                        <div 
                          className="emotion-bar-fill"
                          style={{ 
                            width: `${(count / emotionStats.total) * 100}%`,
                            backgroundColor: emotionColors[emotion]
                          }}
                        ></div>
                      </div>
                    </div>
                  ))
                ) : (
                  <p className="empty-analysis">No emotion data yet</p>
                )}
              </div>
            </div>

            {/* Empathy Growth */}
            <div className="analysis-section">
              <h3>Empathy Growth</h3>
              {empathyGrowth.length > 0 ? (
                <>
                  <div className="trend-indicator">
                    <span className={`trend-value ${improvementTrend >= 0 ? 'positive' : 'negative'}`}>
                      {improvementTrend >= 0 ? '↑' : '↓'} {Math.abs(improvementTrend)}%
                    </span>
                    <span className="trend-label">this week</span>
                  </div>
                  <div className="growth-chart">
                    {empathyGrowth.map((day, index) => (
                      <div key={index} className="chart-bar-container">
                        <div className="chart-bar-label">{day.day}</div>
                        <div className="chart-bar-wrapper">
                          <div 
                            className="chart-bar"
                            style={{ 
                              height: `${day.score}%`,
                              backgroundColor: day.score > 60 ? '#34D399' : day.score > 30 ? '#F59E0B' : '#FF5252'
                            }}
                          ></div>
                        </div>
                        <div className="chart-value">{day.score}%</div>
                      </div>
                    ))}
                  </div>
                </>
              ) : (
                <p className="empty-analysis">Send messages to see your growth</p>
              )}
            </div>

            {/* Common Trigger Words */}
            <div className="analysis-section">
              <h3>Common Trigger Words</h3>
              <div className="trigger-words">
                {triggerWords.length > 0 ? (
                  triggerWords.map(([word, count]) => (
                    <div key={word} className="trigger-word-item">
                      <span className="trigger-word">"{word}"</span>
                      <span className="trigger-count">{count} time{count !== 1 ? 's' : ''}</span>
                      <div className="trigger-bar">
                        <div 
                          className="trigger-bar-fill"
                          style={{ 
                            width: `${(count / messages.length) * 100}%`,
                            backgroundColor: '#FF5252'
                          }}
                        ></div>
                      </div>
                    </div>
                  ))
                ) : (
                  <p className="empty-analysis">No trigger words detected</p>
                )}
              </div>
            </div>

            {/* Quick Stats */}
            <div className="analysis-section stats-grid">
              <h3>Quick Stats</h3>
              <div className="stats-cards">
                <div className="stat-card">
                  <span className="stat-value">{emotionStats.total}</span>
                  <span className="stat-label">Messages</span>
                </div>
                <div className="stat-card">
                  <span className="stat-value">
                    {emotionStats.total > 0 ? Math.round((emotionStats.Positive / emotionStats.total) * 100) : 0}%
                  </span>
                  <span className="stat-label">Positive</span>
                </div>
                <div className="stat-card">
                  <span className="stat-value">
                    {messages.filter(m => m.rewound).length}
                  </span>
                  <span className="stat-label">Rewritten</span>
                </div>
                <div className="stat-card">
                  <span className="stat-value">
                    {mostUsedEmotions[0]?.[0] || 'N/A'}
                  </span>
                  <span className="stat-label">Top Emotion</span>
                </div>
              </div>
            </div>
          </div>
        )}
      </div>

      {/* Info Footer */}
      <div className="info-footer">
        <p>💡 We help you express feelings without hurting relationships.</p>
      </div>
    </div>
  );
}

export default App;




//App.css
body {
  margin: 0;
  font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
  background: linear-gradient(135deg, #c3dafe, #e0c3fc);
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.App {
  background: white;
  padding: 30px;
  border-radius: 24px;
  width: 500px;
  max-width: 100%;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  animation: fadeSlideIn 0.5s ease-out;
}

/* Header */
.header {
  text-align: center;
  margin-bottom: 25px;
}

.header h1 {
  margin: 0 0 5px 0;
  color: #111827;
  font-size: 2rem;
  font-weight: 600;
}

.tagline {
  color: #6b7280;
  font-size: 0.9rem;
  font-style: italic;
  margin: 0 0 15px 0;
}

/* Privacy Toggle */
.privacy-toggle {
  background: #f3f4f6;
  color: #4b5563;
  border: 1px solid #e5e7eb;
  padding: 8px 16px;
  border-radius: 30px;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.privacy-toggle:hover {
  background: #e5e7eb;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.privacy-toggle.active {
  background: #4b5563;
  color: white;
  border-color: #4b5563;
}

/* Input Section */
.input-section {
  display: flex;
  gap: 10px;
  margin-bottom: 25px;
}

.input-section input {
  flex: 1;
  padding: 14px 16px;
  border-radius: 16px;
  border: 2px solid #e5e7eb;
  outline: none;
  font-size: 14px;
  transition: all 0.2s ease;
  background: white;
}

.input-section input:focus {
  border-color: #7f9cf5;
  box-shadow: 0 0 0 3px rgba(127, 156, 245, 0.1);
}

.input-section input:disabled {
  background: #f9fafb;
  cursor: not-allowed;
}

.input-section button {
  padding: 0 24px;
  border: none;
  border-radius: 16px;
  background: #7f9cf5;
  color: white;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.input-section button:hover:not(:disabled) {
  background: #667eea;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(127, 156, 245, 0.2);
}

.input-section button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

/* Tabs */
.tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 20px;
  border-bottom: 2px solid #f3f4f6;
  padding-bottom: 10px;
}

.tab {
  flex: 1;
  padding: 10px;
  border: none;
  background: none;
  color: #9ca3af;
  font-size: 0.95rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  border-radius: 12px;
}

.tab:hover {
  color: #6b7280;
  background: #f9fafb;
}

.tab.active {
  color: #7f9cf5;
  background: #eff6ff;
  font-weight: 600;
}

/* Content Area */
.content-area {
  min-height: 300px;
  max-height: 400px;
  overflow-y: auto;
  margin-bottom: 20px;
  padding-right: 5px;
}

/* Messages List */
.messages-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.message-card {
  background: #f9fafb;
  border-radius: 16px;
  padding: 16px;
  transition: all 0.2s ease;
}

.message-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  font-size: 0.85rem;
}

.sender {
  font-weight: 600;
  color: #374151;
}

.timestamp {
  color: #9ca3af;
}

.message-text {
  margin: 0 0 12px 0;
  color: #1f2937;
  font-size: 0.95rem;
  line-height: 1.5;
}

.message-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.emotion-badge {
  padding: 4px 12px;
  border-radius: 30px;
  color: white;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  display: inline-block;
}

.rewind-btn {
  padding: 4px 12px;
  background: none;
  border: 1px solid #e5e7eb;
  border-radius: 30px;
  color: #6b7280;
  font-size: 0.75rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.rewind-btn:hover {
  background: #f3f4f6;
  border-color: #9ca3af;
}

/* Suggestions List */
.suggestions-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.suggestion-card {
  background: #f9fafb;
  border-radius: 16px;
  padding: 16px;
}

.suggestion-original {
  margin-bottom: 12px;
  padding-bottom: 12px;
  border-bottom: 1px dashed #e5e7eb;
}

.suggestion-original small,
.suggestion-bridged small {
  color: #9ca3af;
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  display: block;
  margin-bottom: 4px;
}

.suggestion-original p {
  margin: 4px 0 8px 0;
  color: #4b5563;
  font-style: italic;
  font-size: 0.9rem;
}

.bridged-text {
  color: #1f2937;
  font-weight: 500;
  margin: 4px 0 12px 0;
  line-height: 1.5;
}

.use-suggestion-btn {
  width: 100%;
  padding: 10px;
  background: white;
  border: 2px solid #7f9cf5;
  color: #7f9cf5;
  border-radius: 12px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.use-suggestion-btn:hover {
  background: #7f9cf5;
  color: white;
}

/* Rewind List */
.rewind-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.rewind-card {
  background: #f9fafb;
  border-radius: 16px;
  padding: 16px;
}

.rewind-comparison {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  gap: 12px;
  align-items: center;
  margin-bottom: 12px;
}

.original-version,
.rewound-version {
  background: white;
  padding: 12px;
  border-radius: 12px;
}

.original-version small,
.rewound-version small {
  color: #9ca3af;
  font-size: 0.7rem;
  text-transform: uppercase;
  display: block;
  margin-bottom: 4px;
}

.original-version p {
  color: #ef4444;
  margin: 0;
  font-size: 0.9rem;
  font-style: italic;
}

.rewound-version p {
  color: #10b981;
  margin: 0;
  font-size: 0.9rem;
  font-weight: 500;
}

.rewind-arrow {
  color: #9f7aea;
  font-size: 1.2rem;
}

.rewind-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

/* Analysis Dashboard */
.analysis-dashboard {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.analysis-section {
  background: #f9fafb;
  border-radius: 16px;
  padding: 16px;
}

.analysis-section h3 {
  margin: 0 0 12px 0;
  color: #374151;
  font-size: 0.95rem;
  font-weight: 600;
}

/* Emotion Bars */
.emotion-bars {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.emotion-bar-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.emotion-bar-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: #4b5563;
}

.emotion-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  display: inline-block;
}

.emotion-count {
  margin-left: auto;
  color: #9ca3af;
  font-size: 0.8rem;
}

.emotion-bar-container {
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
  width: 100%;
}

.emotion-bar-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease;
}

/* Growth Chart */
.trend-indicator {
  text-align: center;
  margin-bottom: 16px;
}

.trend-value {
  font-size: 1.3rem;
  font-weight: bold;
  display: block;
}

.trend-value.positive {
  color: #34D399;
}

.trend-value.negative {
  color: #FF5252;
}

.trend-label {
  color: #6b7280;
  font-size: 0.8rem;
}

.growth-chart {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 4px;
  height: 120px;
}

.chart-bar-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.chart-bar-label {
  font-size: 0.65rem;
  color: #6b7280;
}

.chart-bar-wrapper {
  width: 100%;
  height: 80px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
}

.chart-bar {
  width: 20px;
  border-radius: 6px 6px 0 0;
  transition: height 0.3s ease;
}

.chart-value {
  font-size: 0.65rem;
  font-weight: 600;
  color: #4b5563;
}

/* Trigger Words */
.trigger-words {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.trigger-word-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.trigger-word {
  font-weight: 600;
  color: #1f2937;
  font-size: 0.9rem;
}

.trigger-count {
  font-size: 0.7rem;
  color: #6b7280;
}

.trigger-bar {
  height: 6px;
  background: #e5e7eb;
  border-radius: 3px;
  overflow: hidden;
  width: 100%;
}

.trigger-bar-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s ease;
}

/* Stats Cards */
.stats-grid {
  margin-bottom: 0;
}

.stats-cards {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
}

.stat-card {
  background: white;
  padding: 12px;
  border-radius: 12px;
  text-align: center;
}

.stat-value {
  display: block;
  font-size: 1.3rem;
  font-weight: bold;
  color: #7f9cf5;
  line-height: 1.2;
}

.stat-label {
  font-size: 0.7rem;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

/* Empty States */
.empty-state, .empty-analysis {
  text-align: center;
  color: #9ca3af;
  padding: 30px 20px;
  font-style: italic;
}

/* Modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.3s ease;
}

.modal-content {
  background: white;
  padding: 24px;
  border-radius: 20px;
  width: 350px;
  max-width: 90%;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.3s ease;
}

.cool-down-icon {
  font-size: 2.5rem;
  text-align: center;
  margin-bottom: 10px;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.modal-content h2 {
  margin: 0 0 8px 0;
  color: #111827;
  text-align: center;
  font-size: 1.3rem;
}

.cool-down-message {
  background: #f3f4f6;
  padding: 12px;
  border-radius: 12px;
  font-style: italic;
  color: #4b5563;
  margin: 12px 0;
  font-size: 0.9rem;
}

.timer {
  font-size: 2.5rem;
  font-weight: bold;
  color: #7f9cf5;
  text-align: center;
  margin: 10px 0;
}

.suggestion-text {
  color: #6b7280;
  margin: 8px 0 4px 0;
  font-size: 0.85rem;
}

.bridged-suggestion {
  background: #eff6ff;
  padding: 12px;
  border-radius: 12px;
  color: #1f2937;
  font-weight: 500;
  margin: 0 0 16px 0;
  font-size: 0.9rem;
}

.original-message-block {
  background: #f3f4f6;
  padding: 12px;
  border-radius: 12px;
  margin: 12px 0;
  position: relative;
}

.original-message-block small {
  color: #9ca3af;
  font-size: 0.65rem;
  text-transform: uppercase;
  display: block;
  margin-bottom: 4px;
}

.original-message-block p {
  margin: 0 0 8px 0;
  color: #4b5563;
  font-style: italic;
  font-size: 0.9rem;
  padding-right: 60px;
}

.original-message-block .emotion-badge {
  position: absolute;
  top: 12px;
  right: 12px;
  font-size: 0.65rem;
  padding: 3px 8px;
}

.rewind-input {
  width: 100%;
  padding: 10px;
  border: 2px solid #e5e7eb;
  border-radius: 12px;
  font-size: 0.9rem;
  margin: 12px 0;
  resize: vertical;
  font-family: inherit;
}

.rewind-input:focus {
  outline: none;
  border-color: #7f9cf5;
}

.suggestion-chips {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.chip {
  padding: 6px 12px;
  background: #f3f4f6;
  border: none;
  border-radius: 30px;
  color: #4b5563;
  font-size: 0.8rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.chip:hover {
  background: #e5e7eb;
}

.modal-actions {
  display: flex;
  gap: 8px;
}

.primary-btn {
  flex: 2;
  padding: 10px;
  background: #7f9cf5;
  color: white;
  border: none;
  border-radius: 12px;
  font-weight: 600;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.primary-btn:hover {
  background: #667eea;
  transform: translateY(-2px);
}

.secondary-btn {
  flex: 1;
  padding: 10px;
  background: #f3f4f6;
  color: #4b5563;
  border: none;
  border-radius: 12px;
  font-weight: 600;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.secondary-btn:hover {
  background: #e5e7eb;
}

/* Info Footer */
.info-footer {
  text-align: center;
  padding-top: 16px;
  border-top: 2px solid #f3f4f6;
  color: #6b7280;
  font-size: 0.9rem;
}

.info-footer p {
  margin: 0;
}

/* Animations */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes fadeSlideIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Scrollbar */
.content-area::-webkit-scrollbar {
  width: 6px;
}

.content-area::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 10px;
}

.content-area::-webkit-scrollbar-thumb {
  background: #cbd5e0;
  border-radius: 10px;
}

.content-area::-webkit-scrollbar-thumb:hover {
  background: #a0aec0;
}

/* Mobile Responsive */
@media (max-width: 600px) {
  .App {
    padding: 20px;
  }

  .input-section {
    flex-direction: column;
  }

  .input-section button {
    padding: 14px;
  }

  .tabs {
    flex-wrap: wrap;
  }

  .tab {
    padding: 8px;
    font-size: 0.85rem;
  }

  .rewind-comparison {
    grid-template-columns: 1fr;
  }

  .rewind-arrow {
    transform: rotate(90deg);
    text-align: center;
  }

  .stats-cards {
    grid-template-columns: 1fr;
  }

  .growth-chart {
    height: 100px;
  }

  .chart-bar {
    width: 15px;
  }

  .modal-actions {
    flex-direction: column;
  }
}




