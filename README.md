import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  onAuthStateChanged,
  signInWithCustomToken,
  signOut
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  serverTimestamp, 
  doc, 
  setDoc, 
  getDoc
} from 'firebase/firestore';
import { 
  MapPin, MessageCircle, User, Heart, Send, ChevronLeft, 
  Crown, CheckCircle, Loader2, Wifi, AlertTriangle, X, 
  Lock, Camera, CreditCard, Bell, LogOut, Gift, 
  Edit2, Check, Save, Info, Cigarette, Wine, 
  Briefcase, Clock, DollarSign, Image as ImageIcon, Video
} from 'lucide-react';

// --- 1. CONFIGURATION ---
const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'pro-quo-sweets-v4-stable';
const FREE_LIMIT = 3;
const MAX_IMG_SIZE = 5 * 1024 * 1024; // Increased to 5MB for videos
const STRIPE_PAYMENT_URL = "PLACEHOLDER_LINK"; 

// --- 2. FIREBASE INIT ---
let db = null;
let auth = null;
let firebaseInitialized = false;

try {
  let config = null;
  if (typeof __firebase_config !== 'undefined') {
    config = typeof __firebase_config === 'string' ? JSON.parse(__firebase_config) : __firebase_config;
  }
  if (config && Object.keys(config).length > 0) {
    const app = initializeApp(config);
    auth = getAuth(app);
    db = getFirestore(app);
    firebaseInitialized = true;
  }
} catch (e) {
  console.warn("Offline Mode Active:", e);
}

// --- 3. MOCK DATA ---
const MOCK_USERS = [
  { 
    userId: 'm1', name: 'Sarah', age: 24, city: 'Downtown', avatarId: 1, type: 'profile', 
    email: 'sarah@demo.com', gender: 'Female', bio: 'Lover of coffee and coding.', 
    occupation: 'Student', budget: 'Moderate', availability: 'Weekends', smoking: 'No', drinking: 'Socially' 
  },
  { 
    userId: 'm2', name: 'Mike', age: 28, city: 'North Side', avatarId: 2, type: 'profile', 
    email: 'mike@demo.com', gender: 'Male', bio: 'Adventure seeker looking for a partner.', 
    occupation: 'Engineer', budget: 'Generous', availability: 'Evenings', smoking: 'Socially', drinking: 'Yes' 
  },
  { 
    userId: 'm3', name: 'Jessica', age: 22, city: 'West End', avatarId: 3, type: 'profile', 
    email: 'jess@demo.com', gender: 'Female', bio: 'Art and design enthusiast.', 
    occupation: 'Designer', budget: 'Open', availability: 'Flexible', smoking: 'No', drinking: 'No' 
  },
];

const getSafeDate = (ts) => {
  if (!ts) return new Date();
  if (ts.toDate) return ts.toDate();
  if (ts.seconds) return new Date(ts.seconds * 1000);
  return new Date(ts);
};

// --- 4. COMPONENTS ---

const Toast = ({ message, onClose }) => (
  <div className="fixed top-4 left-4 right-4 z-[100] animate-in slide-in-from-top-2 fade-in duration-300 pointer-events-none">
    <div className="bg-slate-900/90 backdrop-blur-md text-white px-4 py-3 rounded-2xl shadow-xl flex items-center justify-between border border-white/10 mx-auto max-w-sm pointer-events-auto">
      <div className="flex items-center gap-3">
        <div className="bg-rose-500 rounded-full p-1">
          <Check size={12} strokeWidth={3} />
        </div>
        <span className="font-medium text-sm">{message}</span>
      </div>
      <button onClick={onClose} className="text-slate-400 hover:text-white"><X size={16} /></button>
    </div>
  </div>
);

const Logo = ({ size = 'md' }) => {
  const iconSize = size === 'lg' ? 64 : 28;
  const fontSize = size === 'lg' ? 'text-4xl' : 'text-xl';
  return (
    <div className="flex items-center justify-center gap-2.5 group select-none">
      <div className="relative">
        <div className="absolute inset-0 bg-rose-400 rounded-full blur-lg opacity-40 group-hover:opacity-60 transition-opacity"></div>
        <div className="relative bg-gradient-to-br from-rose-500 to-purple-600 p-2.5 rounded-2xl shadow-lg transform group-hover:rotate-12 transition-transform duration-300">
          <Heart className="text-white fill-white" size={iconSize} />
        </div>
      </div>
      <h1 className={`${fontSize} font-black text-transparent bg-clip-text bg-gradient-to-r from-slate-900 to-slate-700 tracking-tight`}>
        Pro Quo Sweets
      </h1>
    </div>
  );
};

const AuthScreen = ({ onLogin, onSignup, error, loading, mode, onForceOffline }) => {
  const [isLogin, setIsLogin] = useState(true);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    if (isLogin) onLogin(email, password);
    else onSignup(email, password);
  };

  return (
    <div className="min-h-screen bg-slate-50 flex items-center justify-center p-6 font-sans relative overflow-hidden">
      <div className="absolute top-[-20%] right-[-10%] w-96 h-96 bg-rose-200 rounded-full blur-3xl opacity-30 animate-pulse"></div>
      <div className="absolute bottom-[-10%] left-[-10%] w-72 h-72 bg-purple-200 rounded-full blur-3xl opacity-30"></div>

      <div className="bg-white/80 backdrop-blur-xl w-full max-w-md p-8 rounded-[2.5rem] shadow-2xl animate-in fade-in zoom-in duration-300 relative z-10 border border-white">
        <div className="mb-8 flex justify-center">
          <Logo size="lg" />
        </div>
        <p className="text-center text-slate-500 mb-8 font-medium">Sweet connections, made simple.</p>

        {error && (
          <div className="bg-red-50 text-red-500 p-4 rounded-2xl text-sm mb-6 flex items-center gap-2 border border-red-100">
            <AlertTriangle size={16} className="shrink-0" /> {error}
          </div>
        )}

        <form onSubmit={handleSubmit} className="space-y-4">
          <div className="space-y-1">
             <label className="text-xs font-bold text-slate-400 ml-4 uppercase tracking-wider">Email</label>
             <input 
               type="email" 
               required
               className="w-full bg-slate-100 border-2 border-transparent p-4 rounded-2xl outline-none focus:bg-white focus:border-rose-500 transition-all font-medium text-slate-700 placeholder:text-slate-300"
               placeholder="name@example.com"
               value={email}
               onChange={(e) => setEmail(e.target.value)}
             />
          </div>
          <div className="space-y-1">
             <label className="text-xs font-bold text-slate-400 ml-4 uppercase tracking-wider">Password</label>
             <input 
               type="password" 
               required
               className="w-full bg-slate-100 border-2 border-transparent p-4 rounded-2xl outline-none focus:bg-white focus:border-rose-500 transition-all font-medium text-slate-700 placeholder:text-slate-300"
               placeholder="••••••••"
               value={password}
               onChange={(e) => setPassword(e.target.value)}
             />
          </div>

          <button 
            disabled={loading}
            className="w-full bg-slate-900 text-white font-bold py-4.5 rounded-2xl shadow-lg shadow-slate-200 hover:bg-slate-800 active:scale-[0.98] transition-all flex justify-center items-center gap-2 mt-4 disabled:opacity-70"
          >
            {loading && <Loader2 className="animate-spin" size={20} />}
            {isLogin ? 'Sign In' : 'Create Account'}
          </button>
        </form>

        <div className="mt-8 text-center">
          <p className="text-slate-400 text-sm">
            {isLogin ? "First time?" : "Returning?"}{" "}
            <button 
              onClick={() => { setIsLogin(!isLogin); setPassword(''); }}
              className="text-rose-500 font-bold hover:text-rose-600 transition-colors"
            >
              {isLogin ? "Join Now" : "Log In"}
            </button>
          </p>
        </div>
        
        {mode === 'loading' && (
           <div className="mt-8 pt-6 border-t border-slate-100 text-center">
              <button onClick={onForceOffline} className="text-xs font-bold text-slate-300 hover:text-slate-500 transition-colors uppercase tracking-widest">
                Force Offline Mode
              </button>
           </div>
        )}
      </div>
    </div>
  );
};

const Avatar = ({ seed, size, customImage, onClick, editable, isPremium }) => {
  const d = { sm: 'w-10 h-10', md: 'w-14 h-14', lg: 'w-20 h-20', xl: 'w-32 h-32' }[size];
  const c = ['bg-rose-100 text-rose-600','bg-violet-100 text-violet-600','bg-emerald-100 text-emerald-600'][seed%3];
  
  return (
    <div 
      onClick={onClick}
      className={`${d} ${c} rounded-full flex items-center justify-center shrink-0 relative group ${editable ? 'cursor-pointer' : ''} ${isPremium ? 'ring-4 ring-amber-400 ring-offset-2' : ''} transition-all shadow-sm`}
    >
      <div className="w-full h-full rounded-full overflow-hidden relative">
        {customImage ? (
          <img src={customImage} alt="Profile" className="w-full h-full object-cover" />
        ) : (
          <div className="w-full h-full flex items-center justify-center">
             <User size={size==='xl'?48:(size==='lg'?28:20)} />
          </div>
        )}
      </div>
      
      {editable && (
        <div className="absolute inset-0 bg-black/40 opacity-0 group-hover:opacity-100 flex items-center justify-center transition-opacity rounded-full">
          <Camera className="text-white drop-shadow-md" size={size === 'xl' ? 24 : 16} />
        </div>
      )}
    </div>
  );
};

// -- Full Profile View Modal --
const UserProfileModal = ({ user, onClose, onChat, onLike }) => (
  <div className="fixed inset-0 z-50 bg-white/95 backdrop-blur-lg flex flex-col animate-in slide-in-from-bottom-10 duration-300 overflow-y-auto font-sans">
    <div className="relative">
      <div className="h-48 bg-gradient-to-br from-rose-400 to-purple-500"></div>
      <button onClick={onClose} className="absolute top-4 left-4 p-2 bg-black/20 hover:bg-black/40 text-white rounded-full backdrop-blur-md transition-all">
        <ChevronLeft size={24} />
      </button>
      <div className="absolute -bottom-16 left-6">
        <Avatar seed={user.avatarId} size="xl" customImage={user.customAvatar} isPremium={user.isPremium} />
      </div>
    </div>
    
    <div className="pt-20 px-6 pb-24">
      <div className="flex justify-between items-start mb-6">
        <div>
          <h2 className="text-3xl font-black text-slate-900">{user.name}, {user.age}</h2>
          <p className="text-slate-500 font-medium flex items-center gap-1"><MapPin size={16} /> {user.city}</p>
        </div>
        <div className="flex gap-2">
          <button onClick={() => onLike(user)} className="p-3 bg-rose-50 text-rose-500 rounded-full hover:scale-110 transition-transform shadow-sm"><Heart fill="currentColor" /></button>
        </div>
      </div>

      <div className="bg-slate-50 p-5 rounded-2xl mb-6 border border-slate-100 shadow-sm">
        <h3 className="text-xs font-bold text-slate-400 uppercase mb-2 tracking-wider">About Me</h3>
        <p className="text-slate-800 leading-relaxed font-medium">{user.bio || "No bio added yet."}</p>
      </div>

      <div className="grid grid-cols-2 gap-3 mb-6">
        <div className="bg-white border p-4 rounded-2xl flex flex-col gap-1 shadow-sm">
          <div className="flex items-center gap-2 text-slate-400 text-xs font-bold uppercase"><Briefcase size={14}/> Occupation</div>
          <div className="font-bold text-slate-800">{user.occupation || "N/A"}</div>
        </div>
        <div className="bg-white border p-4 rounded-2xl flex flex-col gap-1 shadow-sm">
          <div className="flex items-center gap-2 text-slate-400 text-xs font-bold uppercase"><DollarSign size={14}/> Lifestyle</div>
          <div className="font-bold text-slate-800">{user.budget || "N/A"}</div>
        </div>
        <div className="bg-white border p-4 rounded-2xl flex flex-col gap-1 shadow-sm">
          <div className="flex items-center gap-2 text-slate-400 text-xs font-bold uppercase"><Clock size={14}/> Availability</div>
          <div className="font-bold text-slate-800">{user.availability || "Flexible"}</div>
        </div>
        <div className="bg-white border p-4 rounded-2xl flex flex-col gap-1 shadow-sm">
          <div className="flex items-center gap-2 text-slate-400 text-xs font-bold uppercase"><MapPin size={14}/> Meeting Pref</div>
          <div className="font-bold text-slate-800">{user.meetingPref || "Public"}</div>
        </div>
      </div>

      <div className="flex gap-4 mb-8">
        <div className="flex items-center gap-2 text-sm text-slate-600 font-medium bg-slate-50 px-3 py-2 rounded-full border border-slate-100">
          <Cigarette size={16} className="text-slate-400" /> {user.smoking === 'No' ? "Non-smoker" : user.smoking || "Ask me"}
        </div>
        <div className="flex items-center gap-2 text-sm text-slate-600 font-medium bg-slate-50 px-3 py-2 rounded-full border border-slate-100">
          <Wine size={16} className="text-slate-400" /> {user.drinking === 'No' ? "Non-drinker" : user.drinking || "Ask me"}
        </div>
      </div>

      <button onClick={() => { onChat(user); onClose(); }} className="w-full bg-slate-900 text-white font-bold py-4 rounded-2xl shadow-lg hover:scale-[1.02] active:scale-95 transition-all flex items-center justify-center gap-2">
        <MessageCircle size={20} /> Send Message
      </button>
    </div>
  </div>
);

const Onboarding = ({ onComplete }) => {
  const [data, setData] = useState({ 
    name: '', age: '', city: '', bio: '', gender: 'Female', interestedIn: 'Male',
    occupation: '', budget: '', availability: '', meetingPref: '', smoking: 'No', drinking: 'Socially'
  });
  const [image, setImage] = useState(null);
  const [loading, setLoading] = useState(false);
  const fileRef = useRef(null);

  const handleFile = (e) => {
    const f = e.target.files[0];
    if (f) {
      if (f.size > MAX_IMG_SIZE) { alert("File too large"); return; }
      const r = new FileReader();
      r.onload = () => setImage(r.result);
      r.readAsDataURL(f);
    }
  };

  const submit = (e) => {
    e.preventDefault();
    if (!data.name || !data.city) return;
    setLoading(true);
    setTimeout(() => onComplete({ ...data, customAvatar: image }), 800);
  };

  return (
    <div className="min-h-screen bg-slate-50 flex items-center justify-center p-6 font-sans">
      <div className="bg-white w-full max-w-md p-8 rounded-[2rem] shadow-xl border border-slate-100 max-h-[90vh] overflow-y-auto">
        <h2 className="text-2xl font-black text-slate-900 mb-2 text-center">Create Profile</h2>
        <p className="text-slate-500 mb-8 text-center font-medium">Tell us about yourself.</p>
        
        <form onSubmit={submit} className="space-y-5">
          <div className="flex justify-center mb-6">
            <div onClick={() => fileRef.current?.click()} className="relative cursor-pointer group">
              <Avatar seed={0} size="xl" customImage={image} editable />
              <div className="absolute bottom-0 right-0 bg-rose-500 text-white p-2.5 rounded-full border-4 border-white shadow-lg group-hover:scale-110 transition-transform">
                <Camera size={18} />
              </div>
            </div>
            <input type="file" ref={fileRef} className="hidden" accept="image/*" onChange={handleFile} />
          </div>
          
          <div className="space-y-4">
            <input className="w-full bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Display Name" value={data.name} onChange={e=>setData({...data, name: e.target.value})} />
            
            <div className="flex gap-3">
               <input type="number" className="w-1/3 bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Age" value={data.age} onChange={e=>setData({...data, age: e.target.value})} />
               <input className="w-2/3 bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="City" value={data.city} onChange={e=>setData({...data, city: e.target.value})} />
            </div>

            <div className="grid grid-cols-2 gap-3">
              <select className="bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 text-slate-600 font-medium" value={data.gender} onChange={e => setData({...data, gender: e.target.value})}>
                <option value="Male">I'm Male</option>
                <option value="Female">I'm Female</option>
                <option value="Non-binary">Non-binary</option>
              </select>
              <select className="bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 text-slate-600 font-medium" value={data.interestedIn} onChange={e => setData({...data, interestedIn: e.target.value})}>
                <option value="Female">Seek Women</option>
                <option value="Male">Seek Men</option>
                <option value="Everyone">Seek All</option>
              </select>
            </div>

            <textarea className="w-full bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 h-28 font-medium" placeholder="Bio / Description..." value={data.bio} onChange={e=>setData({...data, bio: e.target.value})} />
            <input className="w-full bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Occupation" value={data.occupation} onChange={e=>setData({...data, occupation: e.target.value})} />
            
            <div className="grid grid-cols-2 gap-3">
               <input className="bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Lifestyle/Budget" value={data.budget} onChange={e=>setData({...data, budget: e.target.value})} />
               <input className="bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Availability" value={data.availability} onChange={e=>setData({...data, availability: e.target.value})} />
            </div>

            <input className="w-full bg-slate-50 border p-4 rounded-xl outline-none focus:border-rose-500 font-medium" placeholder="Preferred Meeting Place" value={data.meetingPref} onChange={e=>setData({...data, meetingPref: e.target.value})} />

            <div className="grid grid-cols-2 gap-3">
              <select className="bg-slate-50 border p-4 rounded-xl outline-none text-slate-600 font-medium" value={data.smoking} onChange={e => setData({...data, smoking: e.target.value})}>
                <option value="No">No Smoking</option>
                <option value="Yes">Smoker</option>
                <option value="Socially">Social Smoker</option>
              </select>
              <select className="bg-slate-50 border p-4 rounded-xl outline-none text-slate-600 font-medium" value={data.drinking} onChange={e => setData({...data, drinking: e.target.value})}>
                <option value="No">No Drinking</option>
                <option value="Yes">Drinker</option>
                <option value="Socially">Social Drinker</option>
              </select>
            </div>
          </div>

          <button disabled={loading} className="w-full bg-slate-900 text-white font-bold py-4 rounded-xl shadow-lg hover:bg-slate-800 active:scale-[0.98] transition-all mt-6 disabled:opacity-50">
            {loading ? 'Saving...' : 'Finish Setup'}
          </button>
        </form>
      </div>
    </div>
  );
};

const Paywall = ({ onClose, onUpgrade }) => (
  <div className="fixed inset-0 z-[100] flex items-center justify-center p-4 bg-black/60 backdrop-blur-md animate-in fade-in zoom-in duration-200">
    <div className="bg-white w-full max-w-sm rounded-[2rem] p-8 shadow-2xl relative text-center overflow-hidden">
      <div className="absolute top-0 left-0 w-full h-32 bg-gradient-to-br from-amber-300 to-orange-400 opacity-20"></div>
      <button onClick={onClose} className="absolute top-4 right-4 p-2 bg-white/50 hover:bg-white rounded-full transition-colors"><X size={20} /></button>
      
      <div className="relative z-10">
        <div className="bg-gradient-to-br from-amber-300 to-orange-500 w-20 h-20 rounded-full flex items-center justify-center mx-auto mb-6 shadow-lg shadow-amber-200">
          <Crown className="w-10 h-10 text-white" />
        </div>
        <h2 className="text-3xl font-black text-slate-900 mb-2">Go Pro</h2>
        <p className="text-slate-500 mb-8 font-medium">Break limits. Connect freely.</p>
        <button onClick={onUpgrade} className="w-full bg-slate-900 text-white font-bold py-4 rounded-xl shadow-xl shadow-slate-200 hover:scale-[1.02] active:scale-[0.98] transition-all">
          Upgrade for $4.99/mo
        </button>
      </div>
    </div>
  </div>
);

const Chat = ({ me, them, messages, onSend, onBack }) => {
  const [txt, setTxt] = useState('');
  const endRef = useRef(null);
  const fileInputRef = useRef(null);
  
  useEffect(() => endRef.current?.scrollIntoView({ behavior: 'smooth' }), [messages]);

  const handleSend = (e) => {
    e.preventDefault();
    if (!txt.trim()) return;
    onSend(txt, 'text');
    setTxt('');
  };

  const handleFileSelect = (e) => {
    const file = e.target.files[0];
    if (file) {
      if (file.size > MAX_IMG_SIZE) { alert("File too large (Max 5MB)"); return; }
      const reader = new FileReader();
      reader.onload = () => {
        const type = file.type.startsWith('video') ? 'video' : 'image';
        onSend(reader.result, type);
      };
      reader.readAsDataURL(file);
    }
  };

  return (
    <div className="h-full flex flex-col bg-slate-50 absolute inset-0 z-20">
      <div className="px-4 py-3 bg-white/80 backdrop-blur-md border-b flex items-center gap-3 z-10 sticky top-0">
        <button onClick={onBack} className="p-2 hover:bg-slate-100 rounded-full text-slate-600 transition-colors"><ChevronLeft /></button>
        <Avatar seed={them.avatarId} size="sm" customImage={them.customAvatar} isPremium={them.isPremium} />
        <div className="flex-1">
           <span className="font-bold block text-slate-900">{them.name}</span>
           <span className="text-xs text-green-500 font-bold flex items-center gap-1"><div className="w-1.5 h-1.5 bg-green-500 rounded-full animate-pulse"/> Online</span>
        </div>
      </div>
      
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((m, i) => {
          const isMe = m.senderId === me.userId;
          return (
             <div key={m.id} className={`flex ${isMe ? 'justify-end' : 'justify-start'} animate-in slide-in-from-bottom-2 fade-in duration-300`}>
               <div className={`
                 max-w-[80%] shadow-sm transition-all overflow-hidden
                 ${isMe 
                   ? 'bg-rose-500 text-white rounded-2xl rounded-tr-none' 
                   : 'bg-white text-slate-800 border border-slate-100 rounded-2xl rounded-tl-none'}
               `}>
                 {m.mediaType === 'image' ? (
                   <img src={m.text} className="w-full h-auto max-h-60 object-cover rounded-lg" alt="Sent" />
                 ) : m.mediaType === 'video' ? (
                   <video src={m.text} controls className="w-full h-auto max-h-60 rounded-lg" />
                 ) : (
                   <div className="px-5 py-3 font-medium text-sm">{m.text}</div>
                 )}
               </div>
             </div>
          );
        })}
        <div ref={endRef} />
      </div>
      
      <form onSubmit={handleSend} className="p-3 bg-white border-t flex gap-2 pb-6 md:pb-3 items-center">
        <button type="button" onClick={() => fileInputRef.current?.click()} className="p-3 text-slate-400 hover:text-rose-500 hover:bg-rose-50 rounded-full transition-colors">
          <ImageIcon size={20} />
        </button>
        <input type="file" ref={fileInputRef} className="hidden" accept="image/*,video/*" onChange={handleFileSelect} />
        
        <input 
          className="flex-1 bg-slate-100 border-2 border-transparent focus:border-rose-500 focus:bg-white rounded-full px-6 py-3 outline-none transition-all placeholder:text-slate-400 font-medium" 
          value={txt} 
          onChange={e=>setTxt(e.target.value)} 
          placeholder="Type a message..." 
        />
        <button disabled={!txt} className="bg-rose-500 text-white p-3 rounded-full hover:scale-110 active:scale-95 transition-all shadow-md disabled:opacity-50 disabled:scale-100">
          <Send size={20} className="ml-0.5" />
        </button>
      </form>
    </div>
  );
};

const Profile = ({ profile, onUpgrade, onUpdatePhoto, onRequestNotify, onLogout, onUpdateProfile }) => {
  const [isEditing, setIsEditing] = useState(false);
  const [editData, setEditData] = useState({});
  const fileRef = useRef(null);

  const startEdit = () => {
    setEditData({ ...profile });
    setIsEditing(true);
  };

  const saveEdit = () => {
    onUpdateProfile(editData);
    setIsEditing(false);
  };

  const handlePhotoUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      if (file.size > MAX_IMG_SIZE) { alert("File too large"); return; }
      const reader = new FileReader();
      reader.onloadend = () => onUpdatePhoto(reader.result);
      reader.readAsDataURL(file);
    }
  };

  return (
    <div className="h-full p-6 flex flex-col items-center overflow-y-auto pb-24 bg-slate-50/50">
      <div className="relative group mb-6">
        <Avatar 
          seed={profile.avatarId} 
          size="xl" 
          customImage={profile.customAvatar} 
          editable 
          onClick={() => fileRef.current?.click()}
          isPremium={profile.isPremium}
        />
        <div className="absolute bottom-0 right-0 bg-slate-900 text-white p-2.5 rounded-full shadow-lg border-4 border-white cursor-pointer hover:scale-110 transition-transform">
           <Camera size={18} />
        </div>
        <input type="file" ref={fileRef} className="hidden" accept="image/*" onChange={handlePhotoUpload} />
      </div>

      {isEditing ? (
        <div className="w-full space-y-3 mb-6 animate-in fade-in zoom-in duration-200">
          <input className="w-full p-3 rounded-xl border font-bold text-center text-lg" value={editData.name} onChange={e => setEditData({...editData, name: e.target.value})} placeholder="Name" />
          <input className="w-full p-3 rounded-xl border text-center" value={editData.city} onChange={e => setEditData({...editData, city: e.target.value})} placeholder="City" />
          <textarea className="w-full p-3 rounded-xl border text-sm" value={editData.bio || ''} onChange={e => setEditData({...editData, bio: e.target.value})} placeholder="Bio..." rows={3} />
          
          <input className="w-full p-3 rounded-xl border text-sm" value={editData.occupation || ''} onChange={e => setEditData({...editData, occupation: e.target.value})} placeholder="Occupation" />
          <input className="w-full p-3 rounded-xl border text-sm" value={editData.budget || ''} onChange={e => setEditData({...editData, budget: e.target.value})} placeholder="Lifestyle Budget" />
          <input className="w-full p-3 rounded-xl border text-sm" value={editData.availability || ''} onChange={e => setEditData({...editData, availability: e.target.value})} placeholder="Availability" />
          <input className="w-full p-3 rounded-xl border text-sm" value={editData.meetingPref || ''} onChange={e => setEditData({...editData, meetingPref: e.target.value})} placeholder="Meeting Pref" />
          
          <div className="grid grid-cols-2 gap-2">
             <select className="p-3 border rounded-xl" value={editData.smoking || 'No'} onChange={e=>setEditData({...editData, smoking: e.target.value})}>
                <option value="No">No Smoke</option><option value="Yes">Smoker</option><option value="Socially">Social</option>
             </select>
             <select className="p-3 border rounded-xl" value={editData.drinking || 'No'} onChange={e=>setEditData({...editData, drinking: e.target.value})}>
                <option value="No">No Drink</option><option value="Yes">Drinker</option><option value="Socially">Social</option>
             </select>
          </div>

          <div className="flex gap-2 mt-4">
            <button onClick={() => setIsEditing(false)} className="flex-1 p-3 rounded-xl font-bold bg-slate-200 hover:bg-slate-300">Cancel</button>
            <button onClick={saveEdit} className="flex-1 p-3 rounded-xl font-bold bg-rose-500 text-white hover:bg-rose-600 flex items-center justify-center gap-2"><Save size={16}/> Save</button>
          </div>
        </div>
      ) : (
        <>
          <div className="text-center w-full relative">
             <button onClick={startEdit} className="absolute right-0 top-0 p-2 text-slate-300 hover:text-rose-500 hover:bg-rose-50 rounded-full transition-all"><Edit2 size={18}/></button>
             <h2 className="text-3xl font-black text-slate-900 mb-1">{profile.name}, {profile.age}</h2>
             <p className="text-slate-500 mb-4 font-medium flex items-center justify-center gap-1"><MapPin size={14} className="text-rose-400" /> {profile.city}</p>
             {profile.bio && <p className="text-slate-600 text-sm mb-6 max-w-xs mx-auto italic">"{profile.bio}"</p>}
             
             <div className="grid grid-cols-2 gap-3 mb-6 text-left">
                <div className="bg-white p-3 rounded-xl border shadow-sm">
                   <div className="text-xs text-slate-400 font-bold uppercase">Occupation</div>
                   <div className="font-bold text-slate-800 text-sm">{profile.occupation || "N/A"}</div>
                </div>
                <div className="bg-white p-3 rounded-xl border shadow-sm">
                   <div className="text-xs text-slate-400 font-bold uppercase">Budget</div>
                   <div className="font-bold text-slate-800 text-sm">{profile.budget || "N/A"}</div>
                </div>
             </div>
          </div>
        </>
      )}

      {!profile.isPremium && (
        <div className="w-full bg-slate-900 text-white p-6 rounded-3xl shadow-xl relative overflow-hidden group mb-6 hover:shadow-2xl transition-all cursor-pointer" onClick={onUpgrade}>
          <div className="relative z-10 flex items-center justify-between">
            <div>
              <div className="text-amber-400 font-bold text-sm uppercase tracking-wide mb-1">Go Premium</div>
              <div className="font-black text-2xl">Remove Limits</div>
            </div>
            <div className="bg-white/10 p-3 rounded-full group-hover:scale-110 transition-transform"><Crown className="text-amber-400" /></div>
          </div>
          <div className="absolute inset-0 bg-gradient-to-r from-transparent via-white/5 to-transparent skew-x-12 translate-x-[-100%] group-hover:translate-x-[100%] transition-transform duration-1000"></div>
        </div>
      )}

      <div className="w-full space-y-3">
        <div className="bg-white p-4 rounded-2xl shadow-sm border border-slate-100 flex items-center justify-between">
            <div className="flex items-center gap-3">
               <div className="bg-rose-50 p-2.5 rounded-full text-rose-500"><Bell size={20} /></div>
               <span className="font-bold text-slate-700">Notifications</span>
            </div>
            <button onClick={onRequestNotify} className="text-xs font-bold text-rose-500 hover:bg-rose-50 px-3 py-1.5 rounded-full transition-colors">Enable</button>
        </div>

        <button onClick={onLogout} className="w-full bg-white border border-slate-200 p-4 rounded-2xl shadow-sm flex items-center gap-3 text-slate-600 hover:bg-slate-50 font-bold transition-all active:scale-[0.98]">
            <div className="bg-slate-100 p-2.5 rounded-full text-slate-500"><LogOut size={20} /></div>
            <span>Sign Out</span>
        </button>
      </div>
    </div>
  );
};

export default function App() {
  const [mode, setMode] = useState('loading');
  const [user, setUser] = useState(null);
  const [profile, setProfile] = useState(null);
  const [tab, setTab] = useState('explore');
  const [chatPartner, setChatPartner] = useState(null);
  const [showPaywall, setShowPaywall] = useState(false);
  const [toastMsg, setToastMsg] = useState('');
  const [viewingProfile, setViewingProfile] = useState(null);
  
  const [usersList, setUsersList] = useState([]);
  const [messagesList, setMessagesList] = useState([]);

  // Auth UI State
  const [authError, setAuthError] = useState('');
  const [isAuthLoading, setIsAuthLoading] = useState(false);

  const dbRef = useRef(null);
  const authRef = useRef(null);

  const showToast = (msg) => {
    setToastMsg(msg);
    setTimeout(() => setToastMsg(''), 3000);
  };

  useEffect(() => {
    const init = async () => {
      if (firebaseInitialized) {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
           try { await signInWithCustomToken(auth, __initial_auth_token); } catch(e) {}
        }
        onAuthStateChanged(auth, async (u) => {
          if (u) {
            setUser(u);
            const docRef = doc(db, 'artifacts', APP_ID, 'public', 'data', `profile_${u.uid}`);
            onSnapshot(docRef, (snap) => {
              if (snap.exists()) setProfile(snap.data());
              else setProfile(null);
              setMode('live');
            });
            onSnapshot(collection(db, 'artifacts', APP_ID, 'public', 'data'), (snap) => {
              const uList = [], mList = [];
              snap.forEach(d => {
                 const v = d.data();
                 if (v.type === 'profile' && v.userId !== u.uid) uList.push({ ...v, userId: v.userId || d.id });
                 if (v.type === 'message') mList.push({ ...v, id: d.id });
              });
              setUsersList(uList);
              setMessagesList(mList);
            });
          } else {
            setUser(null);
            setProfile(null);
            setMode('live');
          }
        });
      } else {
        startDemoMode();
      }
    };
    const timer = setTimeout(() => { if (mode === 'loading') startDemoMode(); }, 3000);
    init();
    return () => clearTimeout(timer);
  }, []);

  const startDemoMode = () => {
    setMode('demo');
    setUser(null);
    setUsersList(MOCK_USERS);
  };

  const handleLogin = async (email, password) => {
    setIsAuthLoading(true);
    setAuthError('');
    if (mode === 'live') {
      try {
        await signInWithEmailAndPassword(auth, email, password);
      } catch (err) {
        setAuthError("Invalid email or password");
        if(err.code === 'auth/network-request-failed') {
           setAuthError("Network error. Switching offline.");
           setTimeout(startDemoMode, 1500);
        }
      } finally {
        setIsAuthLoading(false);
      }
    } else {
      setTimeout(() => {
        setUser({ uid: 'demo-user', email });
        setProfile({ name: 'Demo User', age: 25, city: 'Offline City', avatarId: 0, dailyChats: 0, gender: 'Male', interestedIn: 'Female' });
        setIsAuthLoading(false);
      }, 1000);
    }
  };

  const handleSignup = async (email, password) => {
    setIsAuthLoading(true);
    setAuthError('');
    if (password.length < 6) {
      setAuthError("Password must be at least 6 characters");
      setIsAuthLoading(false);
      return;
    }
    if (mode === 'live') {
      try {
        await createUserWithEmailAndPassword(auth, email, password);
      } catch (err) {
        let msg = "Signup failed";
        if(err.code === 'auth/email-already-in-use') msg = "Email already taken";
        setAuthError(msg);
      } finally {
        setIsAuthLoading(false);
      }
    } else {
      setTimeout(() => {
        setUser({ uid: 'demo-user', email });
        setProfile(null);
        setIsAuthLoading(false);
      }, 1000);
    }
  };

  const handleCreateProfile = async (data) => {
    const newProfile = {
      ...data, userId: user.uid, type: 'profile', avatarId: Math.floor(Math.random() * 6),
      isPremium: false, dailyChats: 0, createdAt: new Date(), email: user.email
    };
    if (mode === 'live') await setDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', `profile_${user.uid}`), newProfile);
    else setProfile(newProfile);
  };

  const handleUpdateProfile = async (updates) => {
    if (mode === 'live') await setDoc(doc(db, 'artifacts', APP_ID, 'public', 'data', `profile_${user.uid}`), updates, { merge: true });
    else setProfile({ ...profile, ...updates });
    showToast("Profile updated");
  };

  const handleSendMessage = async (content, type = 'text') => {
    const chatId = [user.uid, chatPartner.userId].sort().join('_');
    const msg = { type: 'message', chatId, text: content, mediaType: type, senderId: user.uid, receiverId: chatPartner.userId, createdAt: mode === 'live' ? serverTimestamp() : new Date() };
    if (mode === 'live') await addDoc(collection(db, 'artifacts', APP_ID, 'public', 'data'), msg);
    else {
      setMessagesList(prev => [...prev, { ...msg, id: Date.now().toString() }]);
      if (type === 'text') {
        setTimeout(() => setMessagesList(p => [...p, { ...msg, text: "Demo Reply!", senderId: chatPartner.userId, receiverId: user.uid, id: Date.now().toString()+'b' }]), 1000);
      }
    }
  };

  if (mode === 'loading') return <div className="flex flex-col items-center justify-center h-screen bg-rose-500 text-white"><Heart className="w-16 h-16 animate-pulse mb-4" fill="white" /><div className="flex items-center gap-2 font-bold"><Loader2 className="animate-spin" /> Starting Pro Quo Sweets...</div><button onClick={startDemoMode} className="mt-8 text-sm underline opacity-80 hover:opacity-100">Use Offline Prototype</button></div>;
  if (!user) return <AuthScreen onLogin={handleLogin} onSignup={handleSignup} error={authError} loading={isAuthLoading} mode={mode} onForceOffline={startDemoMode} />;
  if (!profile) return <Onboarding onComplete={handleCreateProfile} />;

  const filteredUsers = usersList.filter(u => (!profile?.interestedIn || profile.interestedIn === 'Everyone') ? true : u.gender === profile.interestedIn);
  
  // Chat History Logic
  const getHistory = () => {
    const map = new Map();
    messagesList.forEach(m => {
      const pid = m.senderId === user.uid ? m.receiverId : m.receiverId === user.uid ? m.senderId : null;
      if (pid) {
        let partner = usersList.find(u => u.userId === pid);
        if (!partner && mode === 'demo') partner = MOCK_USERS.find(u => u.userId === pid);
        if (partner) {
           const existing = map.get(pid);
           const t = getSafeDate(m.createdAt);
           if (!existing || t > existing.timestamp) {
             map.set(pid, { ...partner, lastMsg: m.mediaType === 'image' ? '📷 Image' : m.text, timestamp: t });
           }
        }
      }
    });
    return Array.from(map.values()).sort((a,b) => b.timestamp - a.timestamp);
  };

  const currentChatMsgs = chatPartner ? messagesList.filter(m => m.chatId === [user.uid, chatPartner.userId].sort().join('_')).sort((a,b) => getSafeDate(a.createdAt) - getSafeDate(b.createdAt)) : [];

  return (
    <div className="flex flex-col h-screen bg-slate-50 max-w-md mx-auto shadow-2xl relative font-sans overflow-hidden">
      {toastMsg && <Toast message={toastMsg} onClose={() => setToastMsg('')} />}
      {viewingProfile && <UserProfileModal user={viewingProfile} onClose={() => setViewingProfile(null)} onChat={(u) => { setViewingProfile(null); setChatPartner(u); }} onLike={() => showToast(`Liked ${viewingProfile.name}!`)} />}
      
      {!chatPartner && (
        <div className="bg-white/90 backdrop-blur-md px-5 py-4 flex justify-between items-center shadow-sm z-10 sticky top-0">
          <Logo size="sm" />
          {profile.isPremium ? <span className="text-xs font-black text-amber-600 bg-amber-100 px-3 py-1.5 rounded-full flex gap-1"><Crown size={14} fill="currentColor"/> PRO</span> : <button onClick={() => setShowPaywall(true)} className="text-xs font-bold text-white bg-slate-900 px-4 py-2 rounded-full hover:bg-slate-700 transition-colors">Go Pro</button>}
        </div>
      )}

      <div className="flex-1 overflow-hidden relative bg-slate-50">
        {chatPartner ? (
          <Chat me={profile} them={chatPartner} messages={currentChatMsgs} onSend={handleSendMessage} onBack={() => setChatPartner(null)} />
        ) : (
          <>
            {tab === 'explore' && (
              <div className="p-4 space-y-4 pb-24 overflow-y-auto h-full bg-slate-50/50">
                {filteredUsers.length===0 && <div className="text-center mt-20 text-slate-400 font-medium">No matches found.</div>}
                {filteredUsers.map(u => (
                  <div key={u.userId} className="bg-white p-4 rounded-[1.5rem] shadow-sm border border-slate-100 flex items-center gap-4 hover:shadow-md transition-all">
                    <Avatar seed={u.avatarId} size="lg" customImage={u.customAvatar} isPremium={u.isPremium} onClick={() => setViewingProfile(u)} editable />
                    <div className="flex-1 min-w-0">
                      <h3 className="font-bold text-lg text-slate-800">{u.name}, {u.age}</h3>
                      <p className="text-slate-500 text-xs flex items-center gap-1 mb-2 font-medium"><MapPin size={10} className="text-rose-400"/> {u.city}</p>
                      <div className="flex gap-2">
                        <button onClick={() => showToast(`Liked ${u.name}!`)} className="p-3 bg-rose-50 text-rose-500 rounded-xl hover:bg-rose-100 active:scale-95 transition-all"><Heart size={20} /></button>
                        <button onClick={() => setViewingProfile(u)} className="flex-1 py-3 bg-slate-100 text-slate-600 font-bold text-sm rounded-xl hover:bg-slate-200 transition-all">View</button>
                        <button onClick={() => setChatPartner(u)} className="flex-1 py-3 bg-slate-900 text-white font-bold text-sm rounded-xl hover:bg-slate-800 active:scale-95 transition-all">Message</button>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            )}
            {tab === 'chats' && (
              <div className="p-4 space-y-3 h-full overflow-y-auto bg-slate-50/50 pb-24">
                 {getHistory().map(c => (
                    <div key={c.userId} onClick={() => setChatPartner(c)} className="bg-white p-3 rounded-xl border border-slate-100 flex items-center gap-4 cursor-pointer hover:bg-rose-50/50 transition-all">
                      <Avatar seed={c.avatarId} size="md" customImage={c.customAvatar} />
                      <div className="flex-1 min-w-0">
                        <h4 className="font-bold text-slate-900">{c.name}</h4>
                        <p className="text-slate-500 text-sm truncate">{c.lastMsg}</p>
                      </div>
                    </div>
                 ))}
                 {getHistory().length === 0 && <div className="text-center text-slate-400 mt-10 font-medium">No recent chats</div>}
              </div>
            )}
            {tab === 'profile' && <Profile profile={profile} onUpgrade={() => setShowPaywall(true)} onUpdatePhoto={(img) => handleUpdateProfile({customAvatar: img})} onRequestNotify={() => showToast("Notifications enabled")} onLogout={() => window.location.reload()} onUpdateProfile={handleUpdateProfile} />}
          </>
        )}
      </div>

      {!chatPartner && (
        <div className="bg-white border-t border-slate-100 p-2 flex justify-around pb-6 md:pb-3 z-10">
          {[
            { id: 'explore', icon: MapPin, label: 'Explore' },
            { id: 'chats', icon: MessageCircle, label: 'Chats' },
            { id: 'profile', icon: User, label: 'Profile' }
          ].map(item => (
            <button key={item.id} onClick={() => setTab(item.id)} className={`flex flex-col items-center gap-1 w-20 p-2 rounded-2xl transition-all ${tab === item.id ? 'text-rose-500 bg-rose-50 font-bold' : 'text-slate-400 font-medium hover:bg-slate-50'}`}>
              <item.icon size={24} strokeWidth={tab === item.id ? 2.5 : 2} />
              <span className="text-[10px]">{item.label}</span>
            </button>
          ))}
        </div>
      )}

      {showPaywall && <Paywall onClose={() => setShowPaywall(false)} onUpgrade={() => { handleUpdateProfile({isPremium: true}); setShowPaywall(false); }} mode={mode} />}
    </div>
  );
}


