import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, query, orderBy, onSnapshot, serverTimestamp, addDoc } from 'firebase/firestore';

// Global variables provided by the Canvas environment for Firebase
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Helper function to simulate normal distribution (replacement for np.random.normal)
function getRandomNormal(mean, stdDev) {
    let u = 0, v = 0;
    while(u === 0) u = Math.random(); // Converting [0,1) to (0,1)
    while(v === 0) v = Math.random();
    let num = Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
    return num * stdDev + mean;
}

// Main App component
function App() {
    // State for Firebase and user authentication
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // State for health data, anomalies, and recommendations
    const [currentHealthData, setCurrentHealthData] = useState(null);
    const [healthHistory, setHealthHistory] = useState([]);
    const [anomalyAlerts, setAnomalyAlerts] = useState([]);
    const [recommendations, setRecommendations] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [message, setMessage] = useState('');

    // Isolation Forest model instance
    const isolationForestRef = useRef(null);
    const [isIsolationForestLoaded, setIsIsolationForestLoaded] = useState(false);


    // Initialize Firebase and authenticate user
    useEffect(() => {
        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const firebaseAuth = getAuth(app);

            setDb(firestore);
            setAuth(firebaseAuth);

            // Listen for auth state changes
            onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    // Sign in anonymously if no token or user
                    if (initialAuthToken) {
                        await signInWithCustomToken(firebaseAuth, initialAuthToken);
                    } else {
                        await signInAnonymously(firebaseAuth);
                    }
                }
                setIsAuthReady(true);
            });
        } catch (error) {
            console.error("Error initializing Firebase:", error);
            setMessage("Failed to initialize the app. Please try again later.");
            setIsLoading(false);
        }
    }, []);

    // Fetch historical data and set up real-time listener
    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        setIsLoading(true);
        const healthMetricsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/health_metrics`);
        // Order by timestamp to get the latest data first
        const q = query(healthMetricsCollectionRef, orderBy('timestamp', 'desc'));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const history = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setHealthHistory(history);
            if (history.length > 0) {
                setCurrentHealthData(history[0]); // Set the latest data as current
            }
            setIsLoading(false);
        }, (error) => {
            console.error("Error fetching health data:", error);
            setMessage("Failed to load health data. Please check your connection.");
            setIsLoading(false);
        });

        // Cleanup listener on component unmount
        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    // Initialize and train Isolation Forest model with simulated data
    useEffect(() => {
        // Ensure IsolationForest is loaded, there's enough data, and the model isn't already initialized
        if (isIsolationForestLoaded && healthHistory.length > 10 && !isolationForestRef.current) {
            console.log("Training Isolation Forest model...");
            // Filter out any records where heart_rate or blood_oxygen might be missing or null
            const dataForTraining = healthHistory
                .filter(d => d.heart_rate != null && d.blood_oxygen != null)
                .map(d => [d.heart_rate, d.blood_oxygen]);

            if (dataForTraining.length > 0) {
                // Access IsolationForest from the global window object
                isolationForestRef.current = new window.IsolationForest({ contamination: 0.05 });
                isolationForestRef.current.fit(dataForTraining);
                console.log("Isolation Forest model trained.");
            } else {
                console.warn("Not enough valid data to train Isolation Forest model.");
            }
        }
    }, [healthHistory, isIsolationForestLoaded]);

    // Simulate real-time data and add to Firestore
    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        const simulateAndAddData = async () => {
            // Simulate new health data using native JS random functions
            const newHeartRate = Math.floor(getRandomNormal(75, 5)); // Mean 75, std dev 5
            const newBloodOxygen = Math.floor(getRandomNormal(97, 1)); // Mean 97, std dev 1
            const activityLevels = ['low', 'moderate', 'high'];
            const newActivityLevel = activityLevels[Math.floor(Math.random() * activityLevels.length)];

            const newRecord = {
                heart_rate: Math.max(50, Math.min(120, newHeartRate)), // Clamp values
                blood_oxygen: Math.max(90, Math.min(100, newBloodOxygen)), // Clamp values
                activity_level: newActivityLevel,
                timestamp: serverTimestamp() // Firestore server timestamp
            };

            try {
                const healthMetricsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/health_metrics`);
                await addDoc(healthMetricsCollectionRef, newRecord);
            } catch (error) {
                console.error("Error adding simulated data to Firestore:", error);
                setMessage("Failed to add new health data.");
            }
        };

        // Simulate data every 5 seconds
        const intervalId = setInterval(simulateAndAddData, 5000);

        // Cleanup interval on component unmount
        return () => clearInterval(intervalId);
    }, [db, userId, isAuthReady]);

    // Anomaly detection and recommendation logic
    useEffect(() => {
        // Only proceed if currentHealthData is available and IsolationForest model is trained
        if (!currentHealthData || !isolationForestRef.current) {
            setAnomalyAlerts([]);
            setRecommendations([]);
            return;
        }

        const detectAndRecommend = () => {
            const { heart_rate, blood_oxygen } = currentHealthData;

            // Ensure heart_rate and blood_oxygen are valid numbers before predicting
            if (typeof heart_rate !== 'number' || typeof blood_oxygen !== 'number') {
                console.warn("Invalid health data for anomaly detection:", currentHealthData);
                setAnomalyAlerts([]);
                setRecommendations(["Waiting for valid health data to generate recommendations."]);
                return;
            }

            // Anomaly Detection
            const prediction = isolationForestRef.current.predict([[heart_rate, blood_oxygen]]);
            const isAnomaly = prediction[0] === -1; // -1 for anomaly, 1 for normal

            let newAnomalyAlerts = [];
            let newRecommendations = [];

            if (isAnomaly) {
                let anomalyMessage = `Anomaly detected: Heart Rate ${heart_rate} bpm, Blood Oxygen ${blood_oxygen}%`;
                newAnomalyAlerts.push(anomalyMessage);

                // Personalized Recommendations based on anomalies
                if (heart_rate > 90) {
                    newRecommendations.push("Your heart rate is elevated. Consider taking a few deep breaths and resting.");
                }
                if (blood_oxygen < 95) {
                    newRecommendations.push("Your blood oxygen is slightly low. Ensure you are in a well-ventilated area and take slow, deep breaths.");
                }
                if (heart_rate < 60) {
                    newRecommendations.push("Your heart rate is lower than usual. If you feel dizzy or unwell, consult a doctor.");
                }
                if (blood_oxygen < 90) {
                    newRecommendations.push("Critical low blood oxygen detected. Seek immediate medical attention.");
                }
                if (newRecommendations.length === 0) {
                    newRecommendations.push("An unusual health pattern detected. Continue monitoring and consult a healthcare professional if concerns persist.");
                }
            } else {
                // If not anomaly, provide general wellness tips
                newRecommendations.push("Your health metrics are currently normal. Keep up your healthy habits!");
                if (currentHealthData.activity_level === 'low') {
                    newRecommendations.push("Consider increasing your activity level for better cardiovascular health.");
                } else if (currentHealthData.activity_level === 'high') {
                    newRecommendations.push("Great activity level! Remember to stay hydrated.");
                }
            }

            setAnomalyAlerts(newAnomalyAlerts);
            setRecommendations(newRecommendations);
        };

        detectAndRecommend();
    }, [currentHealthData]); // Re-run when currentHealthData changes

    return (
        <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4 sm:p-8 font-inter">
            <div className="max-w-4xl mx-auto bg-white rounded-3xl shadow-xl p-6 sm:p-10 border border-gray-200">
                <h1 className="text-4xl font-extrabold text-center text-gray-800 mb-8 tracking-tight">
                    <span className="text-indigo-600">AI Health</span> Monitor
                </h1>

                {userId && (
                    <p className="text-sm text-gray-500 text-center mb-6">
                        User ID: <span className="font-mono bg-gray-100 px-2 py-1 rounded-md text-gray-700 break-all">{userId}</span>
                    </p>
                )}

                {message && (
                    <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-lg relative mb-6" role="alert">
                        <strong className="font-bold">Error!</strong>
                        <span className="block sm:inline"> {message}</span>
                    </div>
                )}

                {/* Loading state for the entire app */}
                {(!isAuthReady || isLoading) && (
                    <div className="flex items-center justify-center py-10">
                        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900 mx-auto mb-4"></div>
                        <p className="text-lg font-semibold text-gray-700 ml-4">Loading health data...</p>
                    </div>
                )}

                {/* Content once loaded */}
                {isAuthReady && !isLoading && (
                    <>
                        {/* Current Health Metrics */}
                        <div className="mb-10">
                            <h2 className="text-2xl font-bold text-gray-700 mb-5 border-b-2 border-indigo-200 pb-2">
                                Current Health Metrics
                            </h2>
                            {currentHealthData ? (
                                <div className="grid grid-cols-1 md:grid-cols-3 gap-6 text-center">
                                    <div className="bg-blue-50 p-6 rounded-xl shadow-md transition-transform transform hover:scale-105">
                                        <p className="text-md text-blue-600 font-semibold mb-2">Heart Rate</p>
                                        <p className="text-4xl font-bold text-blue-800">{currentHealthData.heart_rate || 'N/A'} <span className="text-xl">bpm</span></p>
                                    </div>
                                    <div className="bg-green-50 p-6 rounded-xl shadow-md transition-transform transform hover:scale-105">
                                        <p className="text-md text-green-600 font-semibold mb-2">Blood Oxygen</p>
                                        <p className="text-4xl font-bold text-green-800">{currentHealthData.blood_oxygen || 'N/A'} <span className="text-xl">%</span></p>
                                    </div>
                                    <div className="bg-purple-50 p-6 rounded-xl shadow-md transition-transform transform hover:scale-105">
                                        <p className="text-md text-purple-600 font-semibold mb-2">Activity Level</p>
                                        <p className="text-4xl font-bold text-purple-800 capitalize">{currentHealthData.activity_level || 'N/A'}</p>
                                    </div>
                                </div>
                            ) : (
                                <p className="text-gray-600 text-center">No current health data available. Simulating data...</p>
                            )}
                        </div>

                        {/* Anomaly Alerts */}
                        <div className="mb-10">
                            <h2 className="text-2xl font-bold text-gray-700 mb-5 border-b-2 border-indigo-200 pb-2">
                                Anomaly Alerts
                            </h2>
                            {anomalyAlerts.length > 0 ? (
                                <div className="space-y-4">
                                    {anomalyAlerts.map((alert, index) => (
                                        <div key={index} className="bg-red-50 border-l-4 border-red-500 text-red-800 p-4 rounded-lg shadow-sm flex items-center">
                                            <svg className="h-6 w-6 text-red-500 mr-3" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
                                            </svg>
                                            <p className="font-medium">{alert}</p>
                                        </div>
                                    ))}
                                </div>
                            ) : (
                                <div className="bg-green-50 border-l-4 border-green-500 text-green-800 p-4 rounded-lg shadow-sm flex items-center">
                                    <svg className="h-6 w-6 text-green-500 mr-3" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
                                    </svg>
                                    <p className="font-medium">No anomalies detected.</p>
                                </div>
                            )}
                        </div>

                        {/* Personalized Recommendations */}
                        <div className="mb-10">
                            <h2 className="text-2xl font-bold text-gray-700 mb-5 border-b-2 border-indigo-200 pb-2">
                                Personalized Recommendations
                            </h2>
                            {recommendations.length > 0 ? (
                                <ul className="list-disc list-inside space-y-3 text-gray-700">
                                    {recommendations.map((rec, index) => (
                                        <li key={index} className="bg-yellow-50 p-3 rounded-lg shadow-sm flex items-start">
                                            <svg className="h-5 w-5 text-yellow-600 mr-2 mt-1 flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                                            </svg>
                                            <span>{rec}</span>
                                        </li>
                                    ))}
                                </ul>
                            ) : (
                                <p className="text-gray-600 text-center">Generating recommendations...</p>
                            )}
                        </div>

                        {/* Historical Data (Simplified for display) */}
                        <div>
                            <h2 className="text-2xl font-bold text-gray-700 mb-5 border-b-2 border-indigo-200 pb-2">
                                Recent Health History
                            </h2>
                            {healthHistory.length > 0 ? (
                                <div className="overflow-x-auto rounded-xl shadow-md border border-gray-200">
                                    <table className="min-w-full divide-y divide-gray-200">
                                        <thead className="bg-gray-50">
                                            <tr>
                                                <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider rounded-tl-xl">
                                                    Timestamp
                                                </th>
                                                <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                                    Heart Rate (bpm)
                                                </th>
                                                <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                                                    Blood Oxygen (%)
                                                </th>
                                                <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider rounded-tr-xl">
                                                    Activity
                                                </th>
                                            </tr>
                                        </thead>
                                        <tbody className="bg-white divide-y divide-gray-200">
                                            {healthHistory.slice(0, 5).map((data, index) => ( // Show last 5 records
                                                <tr key={data.id || index} className={index % 2 === 0 ? 'bg-white' : 'bg-gray-50'}>
                                                    <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                                                        {data.timestamp ? new Date(data.timestamp.toDate()).toLocaleString() : 'N/A'}
                                                    </td>
                                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                                        {data.heart_rate}
                                                    </td>
                                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">
                                                        {data.blood_oxygen}
                                                    </td>
                                                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700 capitalize">
                                                        {data.activity_level}
                                                    </td>
                                                </tr>
                                            ))}
                                        </tbody>
                                    </table>
                                </div>
                            ) : (
                                <p className="text-gray-600 text-center">No historical data yet. Please wait for simulation to start.</p>
                            )}
                        </div>

                        <div className="mt-12 p-6 bg-blue-50 rounded-xl text-sm text-blue-800 border border-blue-200 shadow-inner">
                            <p className="font-bold mb-2">Important Notes:</p>
                            <ul className="list-disc list-inside space-y-1">
                                <li>This is a simulated AI-powered health monitoring system for demonstration purposes.</li>
                                <li>Health data is randomly generated and does not reflect real medical conditions.</li>
                                <li>The anomaly detection model (`IsolationForest`) is trained on limited simulated data and is simplified.</li>
                                <li>Recommendations are rule-based and should **NOT** be taken as actual medical advice. Always consult a qualified healthcare professional for any health concerns.</li>
                                <li>Data is stored in Firestore for real-time updates and persistence for the current user session.</li>
                                <li>**Data Privacy:** In a real-world application, robust data privacy (e.g., HIPAA, GDPR compliance), security, and ethical considerations would be paramount. This demo does not include full-scale privacy features.</li>
                            </ul>
                        </div>
                    </>
                )}
            </div>
            {/* Load IsolationForest and Tailwind CSS via script tags */}
            <script src="https://cdn.jsdelivr.net/npm/isolation-forest@1.0.1/dist/isolation-forest.min.js"></script>
            <script src="https://cdn.tailwindcss.com"></script>
            <script
                dangerouslySetInnerHTML={{
                    __html: `
                        tailwind.config = {
                            theme: {
                                extend: {
                                    fontFamily: {
                                        inter: ['Inter', 'sans-serif'],
                                    },
                                },
                            },
                        };
                    `
                }}
            ></script>
        </div>
    );
}

export default App;
