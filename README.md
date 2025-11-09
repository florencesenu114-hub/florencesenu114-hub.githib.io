// src/App.js

import React, { useState } from 'react';
import FocusTimer from './components/FocusTimer';
import FlightBooking from './components/FlightBooking';
import './App.css'; 

function App() {
  // State to manage the active flight session data
  const [activeSession, setActiveSession] = useState(null);

  // Function to start a new flight session
  const startFlight = (sessionDetails) => {
    // sessionDetails: { duration, departure, destination, seatType, category }
    setActiveSession(sessionDetails);
  };

  // Function to end the flight session 
  const endFlight = () => {
    // *** FUTURE STEP: Save statistics, unlock badges, etc. ***
    console.log("Flight session ended. Saving data...");
    setActiveSession(null);
  };

  return (
    <div className="app-container">
      {/* Renders the Timer (Flight) or the Booking/Home screen */}
      {activeSession ? (
        <FocusTimer session={activeSession} onFlightEnd={endFlight} />
      ) : (
        <FlightBooking onStartFlight={startFlight} />
      )}
    </div>
  );
}

export default App;
// src/components/FlightBooking.js

import React, { useState } from 'react';
import './FlightBooking.css';

// Simple data structure for choices
const airportOptions = ['JFK - New York', 'LHR - London', 'NRT - Tokyo', 'GHA - Accra', 'DXB - Dubai'];
const seatTypeOptions = ['Study', 'Work', 'Reading'];
const categoryOptions = ['Assignments', 'Revision', 'Personal Project', 'Learning', 'Misc.'];

function FlightBooking({ onStartFlight }) {
  // Local state for the booking form
  const [duration, setDuration] = useState(60); // Default to 60 minutes
  const [departure, setDeparture] = useState(airportOptions[0]);
  const [destination, setDestination] = useState(airportOptions[1]);
  const [seatType, setSeatType] = useState(seatTypeOptions[0]);
  const [category, setCategory] = useState(categoryOptions[0]);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (departure === destination) {
        alert("Departure and Destination must be different for a successful flight!");
        return;
    }

    // Pass the session details up to App.js to start the timer
    onStartFlight({ 
        duration: duration * 60, // Convert minutes to seconds for the timer
        departure, 
        destination, 
        seatType, 
        category,
        startTime: Date.now()
    });
  };

  return (
    <div className="flight-booking-container">
        {/* --- Home Screen Content --- */}
        <header className="dashboard-header">
            <h1>FocusDeepfocus ✈️</h1>
            <p className="stats-summary">
                **Total Focus Time:** 0h 00m | **Routes Traveled:** 0 (Start your journey!)
            </p>
        </header>

        {/* --- Booking Form --- */}
        <section className="booking-form-card">
            <h2>Book Your Next Flight</h2>
            <form onSubmit={handleSubmit} className="form-grid">
                
                {/* Flight Duration */}
                <label htmlFor="duration">Flight Duration (Minutes):</label>
                <input 
                    id="duration"
                    type="number" 
                    value={duration} 
                    onChange={(e) => setDuration(Math.max(1, parseInt(e.target.value) || 1))}
                    min="1"
                    required
                />

                {/* Departure Airport */}
                <label htmlFor="departure">Departure Airport:</label>
                <select id="departure" value={departure} onChange={(e) => setDeparture(e.target.value)}>
                    {airportOptions.map(airport => <option key={airport} value={airport}>{airport}</option>)}
                </select>

                {/* Destination Airport */}
                <label htmlFor="destination">Destination Airport:</label>
                <select id="destination" value={destination} onChange={(e) => setDestination(e.target.value)}>
                    {airportOptions.map(airport => <option key={airport} value={airport}>{airport}</option>)}
                </select>

                {/* Seat Type (Ambient Sound) */}
                <label>Seat Type (Ambient Sound):</label>
                <div className="seat-type-buttons">
                    {seatTypeOptions.map(type => (
                        <button
                            key={type}
                            type="button"
                            className={`seat-button ${seatType === type ? 'active' : ''}`}
                            onClick={() => setSeatType(type)}
                        >
                            {type}
                        </button>
                    ))}
                </div>
                
                {/* Focus Category */}
                <label htmlFor="category">Flight Category:</label>
                <select id="category" value={category} onChange={(e) => setCategory(e.target.value)}>
                    {categoryOptions.map(cat => <option key={cat} value={cat}>{cat}</option>)}
                </select>
                
                {/* Submit Button */}
                <button type="submit" className="start-flight-btn">
                    <span role="img" aria-label="Takeoff">🛫</span> Start Flight
                </button>
            </form>
        </section>

        {/* --- Placeholder for Today's Flights and other stats --- */}
        <section className="daily-flights-card">
            <h3>Today's Flights</h3>
            <p>No flights scheduled. Book one now!</p>
        </section>
    </div>
  );
}

export default FlightBooking;
// src/components/FocusTimer.js

import React, { useState, useEffect, useRef } from 'react';
import './FocusTimer.css';

// --- Assets and Configuration ---
// These paths must match the files you put in the public/sounds/ folder!
const AMBIENT_SOUND_PATHS = {
    'Study': '/sounds/cabin.mp3',      
    'Work': '/sounds/wind.mp3',        
    'Reading': '/sounds/waves.mp3',    
};

const SOUND_LABELS = {
    'Study': 'Airplane Cabin Noise',
    'Work': 'Soft Wind',
    'Reading': 'Ocean Waves',
};

// Helper function to format seconds into MM:SS
const formatTime = (seconds) => {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = seconds % 60;
    return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
};

function FocusTimer({ session, onFlightEnd }) {
  const [timeRemaining, setTimeRemaining] = useState(session.duration);
  const [isPaused, setIsPaused] = useState(false);
  const [isDistracted, setIsDistracted] = useState(false); 
  const timerRef = useRef(null);
  const audioRef = useRef(null); 

  // --- 1. AMBIENT SOUND LOGIC ---
  useEffect(() => {
    const soundPath = AMBIENT_SOUND_PATHS[session.seatType];
    
    // Create and configure the audio element
    if (soundPath) {
        audioRef.current = new Audio(soundPath);
        audioRef.current.loop = true; 
        audioRef.current.volume = 0.6; 
        
        // Attempt to start playing the sound
        audioRef.current.play().catch(error => {
            console.warn("Autoplay blocked. User needs to click 'Resume Flight' to start ambient sound.", error);
        });
    }

    // Cleanup: Stop the sound when the component unmounts (flight ends)
    return () => {
      if (audioRef.current) {
        audioRef.current.pause();
        audioRef.current = null;
      }
    };
  }, [session.seatType]);
  
  // Pause/Resume audio when the timer state changes
  useEffect(() => {
    if (audioRef.current) {
        if (isPaused || isDistracted) {
            audioRef.current.pause();
        } else {
            audioRef.current.play().catch(() => {}); // Attempt to resume
        }
    }
  }, [isPaused, isDistracted]);


  // --- 2. DEEP FOCUS MODE LOGIC (Tab Change Detection) ---
  useEffect(() => {
    const handleVisibilityChange = () => {
      // If the page is hidden, set distracted state and pause the timer
      if (document.hidden) {
        setIsDistracted(true);
        setIsPaused(true); 
      } else {
        setIsDistracted(false);
      }
    };

    document.addEventListener('visibilitychange', handleVisibilityChange);

    // Cleanup
    return () => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  }, []); 


  // --- 3. TIMER LOGIC ---
  useEffect(() => {
    if (!isPaused && !isDistracted && timeRemaining > 0) {
      timerRef.current = setInterval(() => {
        setTimeRemaining(prevTime => {
          if (prevTime <= 1) {
            clearInterval(timerRef.current);
            onFlightEnd(); 
            return 0;
          }
          return prevTime - 1;
        });
      }, 1000);
    } else {
      clearInterval(timerRef.current);
    }

    return () => clearInterval(timerRef.current);
  }, [isPaused, isDistracted, timeRemaining, onFlightEnd]);

  // Calculations for display
  const completionPercentage = ((session.duration - timeRemaining) / session.duration) * 100;
  const totalDistance = 1000;
  const distanceRemaining = Math.round(totalDistance * (timeRemaining / session.duration));


  return (
    <div className={`focus-timer-container ${isDistracted ? 'distracted-mode' : ''}`}>
      
      {/* DISTRACTION WARNING OVERLAY */}
      {isDistracted && (
        <div className="distraction-overlay">
          <h2>⚠️ WARNING: Deep Focus Mode Broken!</h2>
          <p>You left the cockpit! Return to the app to resume your flight and focus time.</p>
          <button onClick={() => { setIsPaused(false); setIsDistracted(false); }} className="return-focus-btn">
            Return to Focus
          </button>
        </div>
      )}
      
      {/* The main dashboard content, hidden slightly when distracted */}
      <div className="airplane-dashboard">
        <div className="route-info">
            **Route:** {session.departure} <span className="arrow">→</span> {session.destination}
            <span className="category-tag">| Category: {session.category}</span>
        </div>
        
        <div className="timer-display">
            <span className="time-value">{formatTime(timeRemaining)}</span>
            <span className="label">ETA</span>
        </div>
        
        <div className="flight-status">
            **Status:** {isPaused ? 'Holding' : 'Cruising'}
            <span className="distance-info">| Distance Remaining: {distanceRemaining} km</span>
        </div>
      </div>
      
      <div className="flight-map-container">
        <h2>Live Flight Map</h2>
        

        <div className="flight-path-bar">
            <div 
                className="plane-icon" 
                style={{ left: `${completionPercentage}%` }}
                role="img" 
                aria-label="Moving Airplane Icon"
            >
                🛩️
            </div>
            <div className="progress-fill" style={{ width: `${completionPercentage}%` }}></div>
        </div>
        
        <p className="status-message">
            {completionPercentage < 100 ? 
                `In Flight. Current Altitude: ${SOUND_LABELS[session.seatType]}` 
                : 'Landing...'}
        </p>
      </div>

      <div className="control-panel">
        <div className="control-group">
            <h3>Ambient Sound</h3>
            <p>Currently playing: **{SOUND_LABELS[session.seatType]}**</p>
        </div>

        <div className="control-group">
            <h3>Deep Focus Mode</h3>
            <p>Status: **{isDistracted ? 'STOPPED' : 'ACTIVE'}**</p>
            {/* The deep focus toggle is handled by document.hidden, this is a placeholder */}
            <button className="deep-focus-toggle">Browser Focus Required</button>
        </div>

        <div className="action-buttons">
            <button 
                onClick={() => setIsPaused(!isPaused)}
                className="pause-btn"
                disabled={isDistracted} 
            >
                {isPaused ? 'Resume Flight 🟢' : 'Pause Flight ⏸️'}
            </button>
            <button 
                onClick={() => { if(window.confirm('Are you sure you want to land early?')) onFlightEnd(); }}
                className="land-btn"
                disabled={isDistracted} 
            >
                Emergency Landing 🛬
            </button>
        </div>
      </div>
    </div>
  );
}

export default FocusTimer;
