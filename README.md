# Service-point
import React, { useState, useEffect, useRef } from "react";

// Single-file React component (Tailwind CSS assumed available)
// Purpose: Prototype UI for Mistri listing, rotation, booking + hold-wallet simulation.
// This is a frontend-only mock. Replace mock API calls with real endpoints.

export default function MistriBookingPrototype() {
  // mock mistri data
  const initialMistris = [
    { id: 1, name: "Ramesh (Electrician)", rating: 4.7, onTimePct: 0.9, strikes: 0, jobsToday: 1, available: true },
    { id: 2, name: "Suresh (Electrician)", rating: 4.5, onTimePct: 0.85, strikes: 0, jobsToday: 2, available: true },
    { id: 3, name: "Mahesh (Electrician)", rating: 4.2, onTimePct: 0.7, strikes: 1, jobsToday: 0, available: true },
    { id: 4, name: "Raju (Electrician)", rating: 4.9, onTimePct: 0.95, strikes: 0, jobsToday: 3, available: true },
  ];

  const [mistris, setMistris] = useState(initialMistris);
  const [orderLog, setOrderLog] = useState([]);
  const [selected, setSelected] = useState(null);
  const [showBookingModal, setShowBookingModal] = useState(false);
  const [customer, setCustomer] = useState({ name: "", phone: "", area: "" });
  const [holdWallet, setHoldWallet] = useState([]); // holds pending payments
  const [message, setMessage] = useState("");
  const timerRef = useRef(null);

  // Simple trust score calculation
  function trustScore(m) {
    return (m.rating * 0.4 + m.onTimePct * 5 * 0.3 + Math.max(0, 20 - m.strikes * 5) * 0.2) | 0;
  }

  // Sorting: location assumed same; mix of trust + idle priority
  function sortedMistris() {
    // Give priority to idle (jobsToday low) and lower strikes
    return [...mistris]
      .sort((a, b) => {
        // more idle (lower jobsToday) first
        if (a.jobsToday !== b.jobsToday) return a.jobsToday - b.jobsToday;
        // fewer strikes first
        if (a.strikes !== b.strikes) return a.strikes - b.strikes;
        // higher trust next
        return trustScore(b) - trustScore(a);
      })
      .filter(m => m.available);
  }

  // Show rotated list (shuffle daily) - simple rotation example
  useEffect(() => {
    // rotate once on mount to simulate daily change
    setMistris(prev => {
      const rotated = [...prev];
      rotated.push(rotated.shift());
      return rotated;
    });
  }, []);

  // Booking flow: customer fills, pays -> holdWallet gets entry -> assignment & timer starts
  function openBooking(m) {
    setSelected(m);
    setShowBookingModal(true);
    setMessage("");
  }

  function payAndBook() {
    if (!customer.name || !customer.phone || !selected) {
      setMessage("कृपया नाम, फोन और मिस्त्री चुनें।");
      return;
    }

    // create a booking object
    const booking = {
      id: Date.now(),
      mistriId: selected.id,
      customer: { ...customer },
      status: "pending",
      createdAt: new Date().toISOString(),
      holdAmount: 500, // example
    };

    // add to hold wallet
    setHoldWallet(h => [...h, booking]);
    setOrderLog(log => [...log, { ...booking, note: 'Payment held' }]);
    setShowBookingModal(false);

    // start assignment + accept timer (10 seconds in prototype = 10 min in real)
    startAcceptTimer(booking);
    setMessage("Payment hold में है। मिस्त्री को notify किया गया है।");
  }

  // Timer sim (10s accept, 20s arrival, 40s completion) - for prototype
  function startAcceptTimer(booking) {
    let t = 10; // seconds to accept
    const key = booking.id;

    timerRef.current = setInterval(() => {
      t -= 1;
      if (t <= 0) {
        // check simulated mistri response randomly
        const willAccept = Math.random() > 0.3; // 70% chance to accept
        clearInterval(timerRef.current);
        if (willAccept) {
          // accepted -> start arrival timer
          markBookingAccepted(key);
        } else {
          // reject -> reassign
          markBookingNoShow(key);
        }
      }
    }, 1000);
  }

  function markBookingAccepted(bookingId) {
    setOrderLog(log => log.map(b => b.id === bookingId ? { ...b, status: 'accepted' } : b));
    setMessage('मिस्त्री ने booking accept कर ली है — arrival timer चल रहा है');
    // arrival timer
    setTimeout(() => {
      // simulated arrival success 85%
      if (Math.random() > 0.15) {
        markBookingArrived(bookingId);
      } else {
        markBookingNoShow(bookingId);
      }
    }, 20000);
  }

  function markBookingArrived(bookingId) {
    setOrderLog(log => log.map(b => b.id === bookingId ? { ...b, status: 'arrived' } : b));
    setMessage('मिस्त्री पहुंच गया। Customer से OTP verify करें।');
    // simulate OTP verify + completion
    setTimeout(() => {
      markBookingCompleted(bookingId);
    }, 20000);
  }

  function markBookingCompleted(bookingId) {
    // release hold -> payout
    setOrderLog(log => log.map(b => b.id === bookingId ? { ...b, status: 'completed' } : b));
    setHoldWallet(h => h.filter(b => b.id !== bookingId));
    setMessage('Job complete — payment released to mistri (simulated)');
    // increase jobsToday for mistri
    setMistris(prev => prev.map(m => m.id === (orderLog.find(o => o.id === bookingId)||{}).mistriId ? { ...m, jobsToday: m.jobsToday + 1 } : m));
  }

  function markBookingNoShow(bookingId) {
    setOrderLog(log => log.map(b => b.id === bookingId ? { ...b, status: 'no-show' } : b));
    // penalize the mistri
    const booking = holdWallet.find(h => h.id === bookingId);
    if (booking) {
      setMistris(prev => prev.map(m => m.id === booking.mistriId ? { ...m, strikes: m.strikes + 1, available: m.strikes + 1 < 3 } : m));
    }

    // remove hold and auto-reassign or refund
    setHoldWallet(h => h.filter(b => b.id !== bookingId));
    setMessage('No-show detected — hold cancelled. Auto-reassigning / refunding.');

    // attempt reassign to next candidate
    const candidates = sortedMistris().filter(mm => mm.id !== booking?.mistriId);
    if (candidates.length > 0) {
      const next = candidates[0];
      // auto-create new booking to next
      const newBooking = { ...booking, id: Date.now() + 1, mistriId: next.id, status: 'pending', createdAt: new Date().toISOString() };
      setHoldWallet(h => [...h, newBooking]);
      setOrderLog(log => [...log, { ...newBooking, note: 'Auto reassigned' }]);
      startAcceptTimer(newBooking);
    } else {
      setOrderLog(log => [...log, { id: Date.now(), note: 'No candidate, refund processed' }]);
    }
  }

  return (
    <div className="p-6 max-w-5xl mx-auto">
      <header className="mb-6">
        <h1 className="text-2xl font-bold">Mistri Booking Prototype</h1>
        <p className="text-sm text-gray-600">Demo UI: rotation, hold-wallet, auto reassign, strikes</p>
      </header>

      <section className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div className="col-span-2">
          <div className="bg-white p-4 rounded shadow">
            <h2 className="font-semibold mb-3">Available Mistris</h2>
            <ul>
              {sortedMistris().map(m => (
                <li key={m.id} className="flex justify-between items-center p-2 border-b">
                  <div>
                    <div className="font-medium">{m.name}</div>
                    <div className="text-xs text-gray-500">Rating: {m.rating} | On-time: {Math.round(m.onTimePct*100)}% | Strikes: {m.strikes} | Jobs Today: {m.jobsToday}</div>
                  </div>
                  <div className="flex gap-2">
                    <button className="px-3 py-1 bg-blue-600 text-white rounded" onClick={() => openBooking(m)}>Book</button>
                    <button className="px-3 py-1 border rounded" onClick={() => alert('Profile view (mock)')}>View</button>
                  </div>
                </li>
              ))}
            </ul>
          </div>

          <div className="bg-white p-4 rounded shadow mt-4">
            <h2 className="font-semibold mb-3">Order Log</h2>
            <ul className="text-sm text-gray-700">
              {orderLog.slice().reverse().map(o => (
                <li key={o.id} className="border-b p-2">
                  <div><strong>{o.customer?.name || o.note || 'System'}</strong> — {o.status || 'info'}</div>
                  <div className="text-xs text-gray-500">{o.note || ''} {o.createdAt ? new Date(o.createdAt).toLocaleTimeString() : ''}</div>
                </li>
              ))}
            </ul>
          </div>
        </div>

        <aside className="bg-white p-4 rounded shadow">
          <h2 className="font-semibold mb-3">Booking / Customer</h2>
          <input className="w-full p-2 border mb-2 rounded" placeholder="नाम" value={customer.name} onChange={e => setCustomer(s => ({...s, name: e.target.value}))} />
          <input className="w-full p-2 border mb-2 rounded" placeholder="फोन" value={customer.phone} onChange={e => setCustomer(s => ({...s, phone: e.target.value}))} />
          <input className="w-full p-2 border mb-2 rounded" placeholder="एरिया" value={customer.area} onChange={e => setCustomer(s => ({...s, area: e.target.value}))} />

          <div className="mt-3">
            <div className="text-xs text-gray-500 mb-2">Hold Wallet (pending payments)</div>
            <ul className="text-sm">
              {holdWallet.map(h => (
                <li key={h.id} className="border p-2 rounded mb-2">{h.customer.name} — ₹{h.holdAmount} — {h.status || 'pending'}</li>
              ))}
            </ul>
          </div>

          <div className="mt-4">
            <div className="text-sm text-red-600 mb-2">{message}</div>
            <button className="w-full py-2 bg-green-600 text-white rounded" onClick={() => {
              if (!selected) return setMessage('कृपया पहले मिस्त्री चुनें (Book बटन)');
              payAndBook();
            }}>Pay & Book (Hold ₹500)</button>
          </div>

          <div className="mt-4 text-xs text-gray-500">
            Note: This is a frontend prototype. Replace simulated timers and random accept with real push notifications / mistri responses.
          </div>
        </aside>
      </section>

      {/* Booking modal (simple) */}
      {showBookingModal && selected && (
        <div className="fixed inset-0 bg-black/40 flex items-center justify-center">
          <div className="bg-white p-6 rounded w-96">
            <h3 className="font-semibold mb-2">Book {selected.name}</h3>
            <p className="text-sm text-gray-600 mb-4">Hold payment model active — payment will be held until completion.</p>
            <div className="flex gap-2">
              <button className="px-3 py-1 bg-gray-200 rounded" onClick={() => setShowBookingModal(false)}>Cancel</button>
              <button className="px-3 py-1 bg-blue-600 text-white rounded" onClick={() => { setShowBookingModal(false); }}>Proceed to Pay</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
