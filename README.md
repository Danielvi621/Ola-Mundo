
import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, collection, doc, onSnapshot, updateDoc, setDoc, query } from 'firebase/firestore';
import { 
  Gift, ShoppingBag, Info, X, Heart, Check, Copy, 
  QrCode, ArrowRight, Bed, Utensils, Zap, ChevronLeft, 
  Palette, ShowerHead, Armchair, Star, Sparkles, Clock
} from 'lucide-react';

// Configuração do Firebase utilizando as variáveis de ambiente fornecidas
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'daniel-eduarda-sicilian-final';

// Elementos Decorativos Animados
const FloatingElement = ({ children, className, delay = "0s" }) => (
  <div className={`absolute pointer-events-none animate-float ${className}`} style={{ animationDelay: delay }}>
    {children}
  </div>
);

const LemonSVG = () => (
  <svg viewBox="0 0 100 100" className="w-16 h-16 md:w-24 md:h-24 opacity-20">
    <circle cx="50" cy="50" r="30" fill="#FACC15" />
    <path d="M50 20C50 20 55 10 70 15" stroke="#4D7C0F" strokeWidth="3" strokeLinecap="round" />
    <path d="M45 45Q50 40 55 45" stroke="white" strokeWidth="1" strokeLinecap="round" opacity="0.5"/>
  </svg>
);

const LeafSVG = () => (
  <svg viewBox="0 0 100 100" className="w-12 h-12 md:w-20 md:h-20 opacity-10">
    <path d="M10 90C10 90 40 80 50 50C60 20 90 10 90 10M90 10C90 10 80 40 50 50C20 60 10 90 10 90" fill="#4D7C0F" />
  </svg>
);

const CATEGORIES = [
  { id: 'Cozinha', name: 'Cozinha', icon: Utensils, description: 'Eletroportáteis' },
  { id: 'Quarto', name: 'Quarto', icon: Bed, description: 'Conforto & Cama' },
  { id: 'Sala', name: 'Sala', icon: Armchair, description: 'Estar & Estilo' },
  { id: 'Banheiro', name: 'Banheiro', icon: ShowerHead, description: 'Banho & Spa' },
  { id: 'Eletros', name: 'Eletros', icon: Zap, description: 'Tecnologia' },
  { id: 'Decoração', name: 'Decoração', icon: Palette, description: 'Toque Final' }
];

const INITIAL_GIFTS = [
  { id: 'f1', title: 'Geladeira Frost Free Brastemp Inox', price: 'R$ 2.890,00', category: 'Cozinha', image: 'https://images.unsplash.com/photo-1571175452281-04a1fb1f24d9?q=80&w=800&auto=format&fit=crop', description: 'Design moderno com painel eletrônico externo.' },
  { id: 'f2', title: 'Jogo de Panelas Le Creuset Amarelo', price: 'R$ 2.450,00', category: 'Cozinha', image: 'https://images.unsplash.com/photo-1584990344616-3b94b305740a?q=80&w=800&auto=format&fit=crop', description: 'Tradição e performance no tom Limão Siciliano.' },
  { id: 'f3', title: 'Aparelho de Jantar Majólica 42pçs', price: 'R$ 1.180,00', category: 'Cozinha', image: 'https://images.unsplash.com/photo-1610701596007-11502861dcfa?q=80&w=800&auto=format&fit=crop', description: 'Estampa clássica mediterrânica em azul e branco.' },
  { id: 'f4', title: 'Mesa de Centro Madeira Natural', price: 'R$ 1.450,00', category: 'Sala', image: 'https://images.unsplash.com/photo-1533090161767-e6ffed986c88?q=80&w=800&auto=format&fit=crop', description: 'Minimalismo orgânico para o centro da nossa sala.' },
  { id: 'f5', title: 'Tapete Artesanal Azul Navy', price: 'R$ 1.850,00', category: 'Sala', image: 'https://images.unsplash.com/photo-1575414003591-ece8d0416c7a?q=80&w=800&auto=format&fit=crop', description: 'Toque macio com fibras naturais de alta qualidade.' },
  { id: 'f6', title: 'Kit Banho Spa Buddemeyer Luxo', price: 'R$ 620,00', category: 'Banheiro', image: 'https://images.unsplash.com/photo-1560343776-97e7d202ff0e?q=80&w=800&auto=format&fit=crop', description: 'Toalhas em algodão egípcio de extrema absorção.' },
  { id: 'f7', title: 'Jogo de Cama 1000 Fios King', price: 'R$ 1.350,00', category: 'Quarto', image: 'https://images.unsplash.com/photo-1522771739844-6a9f6d5f14af?q=80&w=800&auto=format&fit=crop', description: 'A suavidade do puro algodão para o nosso descanso.' },
  { id: 'f8', title: 'Smart TV 55" 4K Samsung Crystal', price: 'R$ 2.680,00', category: 'Eletros', image: 'https://images.unsplash.com/photo-1593784991095-a205029471b6?q=80&w=800&auto=format&fit=crop', description: 'Resolução incrível para nossos momentos de cinema.' }
];

export default function App() {
  const [user, setUser] = useState(null);
  const [gifts, setGifts] = useState([]);
  const [view, setView] = useState('home');
  const [activeCategory, setActiveCategory] = useState(null);
  const [selectedGift, setSelectedGift] = useState(null);
  const [guestName, setGuestName] = useState('');
  const [pixStep, setPixStep] = useState(false);
  const [copied, setCopied] = useState(false);
  const [showSuccess, setShowSuccess] = useState(false);
  const [daysLeft, setDaysLeft] = useState(0);

  const weddingDate = new Date('2026-06-09T00:00:00');
  const pixKey = "62998808828";

  useEffect(() => {
    const calculate = () => {
      const now = new Date();
      const diff = weddingDate.getTime() - now.getTime();
      setDaysLeft(Math.ceil(diff / (1000 * 60 * 60 * 24)));
    };
    calculate();
    const interval = setInterval(calculate, 3600000);
    return () => clearInterval(interval);
  }, []);

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) { console.error("Erro na autenticação:", err); }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  useEffect(() => {
    if (!user) return;
    const giftsRef = collection(db, 'artifacts', appId, 'public', 'data', 'gifts');
    const unsubscribe = onSnapshot(query(giftsRef), (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      if (data.length === 0) {
        INITIAL_GIFTS.forEach(g => setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'gifts', g.id), { ...g, claimedBy: null }));
      } else {
        setGifts(data);
      }
    });
    return () => unsubscribe();
  }, [user]);

  const confirmReservation = async () => {
    if (!guestName.trim() || !selectedGift) return;
    const giftRef = doc(db, 'artifacts', appId, 'public', 'data', 'gifts', selectedGift.id);
    try {
      await updateDoc(giftRef, { claimedBy: guestName, claimedAt: new Date().toISOString() });
      setSelectedGift(null);
      setPixStep(false);
      setShowSuccess(true);
      setGuestName('');
      setTimeout(() => setShowSuccess(false), 5000);
    } catch (err) { console.error("Erro ao atualizar:", err); }
  };

  const copyPix = () => {
    const el = document.createElement('textarea');
    el.value = pixKey;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  const availableGifts = gifts.filter(g => !g.claimedBy);
  const currentViewGifts = useMemo(() => {
    if (!activeCategory) return [];
    return availableGifts.filter(g => g.category === activeCategory.id);
  }, [availableGifts, activeCategory]);

  return (
    <div className="min-h-screen bg-[#FDFCFB] text-[#1B365D] font-serif relative overflow-x-hidden selection:bg-[#FACC15]/30">

      {/* Elementos Decorativos de Fundo */}
      <FloatingElement className="-left-10 top-20" delay="0s"><LemonSVG /></FloatingElement>
      <FloatingElement className="-right-10 top-60" delay="2s"><LemonSVG /></FloatingElement>
      <FloatingElement className="left-10 bottom-40" delay="4s"><LeafSVG /></FloatingElement>
      <FloatingElement className="right-20 bottom-10" delay="1s"><LeafSVG /></FloatingElement>

      {/* Header Principal */}
      <header className="relative pt-16 pb-16 text-center px-6 z-20 bg-white/60 backdrop-blur-sm border-b-[6px] border-[#1B365D] shadow-lg">
        <div className="mb-8">
          <div className="inline-flex items-center space-x-3 bg-[#1B365D] text-[#FACC15] px-8 py-3 rounded-full shadow-2xl animate-pulse">
            <Clock size={16} />
            <span className="text-[11px] uppercase tracking-[0.2em] font-sans font-black">
              Faltam {daysLeft} dias para o Sim!
            </span>
          </div>
        </div>

        <div className="flex items-center justify-center space-x-3 mb-6">
          <div className="h-px w-12 bg-[#FACC15]"></div>
          <Heart className="w-5 h-5 fill-[#FACC15] text-[#FACC15]" />
          <div className="h-px w-12 bg-[#FACC15]"></div>
        </div>

        <h1 className="text-5xl md:text-8xl font-light tracking-tighter mb-4 text-[#1B365D]">
          Daniel <span className="text-[#FACC15] italic">&</span> Eduarda
        </h1>
        <p className="text-[12px] uppercase tracking-[0.6em] text-[#1B365D]/50 font-sans font-black">09 de Junho de 2026</p>
      </header>

      {/* Conteúdo Principal */}
      <main className="max-w-4xl mx-auto px-6 py-16 relative z-10">
        
        {view === 'home' ? (
          <div className="animate-in fade-in slide-in-from-bottom-12 duration-1000">
            <div className="text-center mb-16">
              <h2 className="text-2xl md:text-3xl font-serif text-[#1B365D] mb-4 flex items-center justify-center space-x-3">
                <Sparkles size={20} className="text-[#FACC15]" />
                <span>Nossa Lista de Presentes</span>
                <Sparkles size={20} className="text-[#FACC15]" />
              </h2>
              <div className="h-1 w-12 bg-[#FACC15] mx-auto mb-4"></div>
              <p className="text-[#1B365D]/60 italic font-sans text-sm text-center">Toque num balão para ver os itens do departamento</p>
            </div>

            {/* Grelha 2x2 Categorias */}
            <div className="grid grid-cols-2 gap-6 md:gap-12 max-w-2xl mx-auto">
              {CATEGORIES.map((cat) => (
                <button
                  key={cat.id}
                  onClick={() => { setActiveCategory(cat); setView('category'); window.scrollTo({ top: 0, behavior: 'smooth' }); }}
                  className="group relative flex flex-col items-center justify-center aspect-square bg-white border-2 border-[#1B365D] rounded-[50px] md:rounded-[80px] p-6 transition-all duration-500 hover:bg-[#FDF9F0] hover:border-[#FACC15] hover:-translate-y-3 shadow-xl active:scale-95"
                >
                  <div className="w-16 h-16 md:w-24 md:h-24 rounded-full bg-[#1B365D] text-[#FACC15] flex items-center justify-center mb-6 transition-all group-hover:scale-110 group-hover:rotate-6 shadow-xl">
                    <cat.icon size={32} strokeWidth={1.5} />
                  </div>
                  <h3 className="text-lg md:text-2xl font-serif text-[#1B365D] mb-1">{cat.name}</h3>
                  <p className="text-[9px] md:text-[11px] uppercase tracking-widest text-[#1B365D]/40 font-sans font-black">
                    {cat.description}
                  </p>
                </button>
              ))}
            </div>
          </div>
        ) : (
          <div className="animate-in slide-in-from-right-12 duration-700">
            {/* Header da Subpágina com Linhas de Fundo */}
            <div className="flex flex-col md:flex-row items-center justify-between mb-12 bg-[#1B365D] p-8 md:p-12 rounded-[50px] text-white shadow-2xl relative overflow-hidden pattern-blue">
              <button 
                onClick={() => setView('home')} 
                className="flex items-center space-x-3 text-[#FACC15] hover:text-white transition-colors group px-6 py-3 border border-[#FACC15]/30 rounded-full mb-6 md:mb-0 relative z-10"
              >
                <ChevronLeft size={20} className="group-hover:-translate-x-1 transition-transform" />
                <span className="text-[11px] uppercase tracking-widest font-bold font-sans">Voltar ao Início</span>
              </button>
              <div className="text-center md:text-right relative z-10">
                <h2 className="text-3xl md:text-5xl font-serif text-[#FACC15] mb-2">{activeCategory.name}</h2>
                <div className="flex items-center justify-center md:justify-end space-x-2">
                   <div className="h-2 w-2 rounded-full bg-[#FACC15] animate-ping"></div>
                   <span className="text-[10px] uppercase tracking-widest font-black text-white/80">Disponíveis: {currentViewGifts.length}</span>
                </div>
              </div>
            </div>

            {/* Grelha 2x2 Itens */}
            {currentViewGifts.length > 0 ? (
              <div className="grid grid-cols-2 gap-6 md:gap-12">
                {currentViewGifts.map((gift) => (
                  <div key={gift.id} className="bg-white border-2 border-[#1B365D]/5 hover:border-[#1B365D] transition-all duration-500 rounded-[40px] overflow-hidden flex flex-col group shadow-md hover:shadow-2xl hover:-translate-y-2">
                    <div className="relative aspect-square overflow-hidden bg-[#F7F3F0]">
                      <img src={gift.image} alt={gift.title} className="w-full h-full object-cover transition-transform duration-[2.5s] group-hover:scale-110" />
                      <div className="absolute top-4 left-4 bg-[#1B365D] text-[#FACC15] px-4 py-2 text-[9px] md:text-[11px] font-black font-sans tracking-widest rounded-full shadow-lg">
                        {gift.price}
                      </div>
                    </div>
                    <div className="p-6 md:p-10 text-center flex flex-col flex-grow">
                      <h3 className="text-lg md:text-2xl font-serif text-[#1B365D] mb-4 h-12 flex items-center justify-center leading-tight">{gift.title}</h3>
                      <p className="text-[10px] md:text-[12px] text-[#1B365D]/50 font-sans italic leading-relaxed mb-8 flex-grow line-clamp-3">{gift.description}</p>
                      <button 
                        onClick={() => { setSelectedGift(gift); setPixStep(false); }}
                        className="w-full py-4 bg-[#FACC15] text-[#1B365D] text-[11px] uppercase tracking-[0.3em] font-black rounded-full hover:bg-[#1B365D] hover:text-white transition-all duration-300 font-sans shadow-lg active:scale-95"
                      >
                        Presentear
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            ) : (
              <div className="py-32 text-center bg-white border-2 border-dashed border-[#1B365D]/20 rounded-[60px] shadow-inner">
                <Check className="w-12 h-12 text-[#FACC15] mx-auto mb-6" />
                <h3 className="text-2xl font-serif text-[#1B365D]">Tudo Reservado!</h3>
                <p className="text-[#1B365D]/40 text-sm mt-2 italic font-sans">Agradecemos de coração pelo carinho.</p>
              </div>
            )}
          </div>
        )}
      </main>

      {/* Modal Unificado */}
      {selectedGift && (
        <div className="fixed inset-0 z-50 flex items-center justify-center p-6 bg-[#1B365D]/70 backdrop-blur-md">
          <div className="bg-white max-w-sm w-full p-10 md:p-12 relative animate-in zoom-in-95 duration-500 rounded-[50px] border-b-[10px] border-[#FACC15] shadow-2xl">
            <button onClick={() => setSelectedGift(null)} className="absolute top-8 right-10 text-[#1B365D]/20 hover:text-[#1B365D] transition-colors"><X size={24} /></button>
            
            {!pixStep ? (
              <div className="text-center font-sans animate-in fade-in">
                <div className="w-16 h-16 bg-[#FDF9F0] rounded-full flex items-center justify-center mx-auto mb-6 shadow-inner">
                  <ShoppingBag size={28} className="text-[#FACC15]" />
                </div>
                <h2 className="text-2xl font-serif text-[#1B365D] mb-2 tracking-tight text-center">Finalizar Escolha</h2>
                <p className="text-[10px] uppercase tracking-[0.3em] text-[#1B365D]/40 font-black mb-10 text-center">{selectedGift.title}</p>
                
                <div className="space-y-8">
                  <div className="text-left border-b-2 border-[#1B365D]/5 pb-1">
                    <label className="text-[9px] uppercase tracking-widest text-[#1B365D]/40 font-black block mb-2">Seu Nome / Família</label>
                    <input 
                      type="text" value={guestName} onChange={(e) => setGuestName(e.target.value)}
                      placeholder="Ex: Família Rossi"
                      className="w-full py-2 outline-none text-base text-[#1B365D] bg-transparent font-medium"
                    />
                  </div>
                  <div className="space-y-4 pt-4">
                    <button onClick={() => setPixStep(true)} className="w-full py-5 bg-[#1B365D] text-[#FACC15] text-[11px] uppercase tracking-[0.2em] font-black rounded-full shadow-xl hover:scale-[1.02] active:scale-95 transition-all relative overflow-hidden pattern-blue">Presentear via PIX</button>
                    <button onClick={confirmReservation} disabled={!guestName.trim()} className="w-full py-5 border-2 border-[#1B365D]/10 text-[#1B365D] text-[11px] uppercase tracking-[0.2em] font-black rounded-full hover:bg-stone-50 disabled:opacity-20 transition-all">Apenas Reservar</button>
                  </div>
                </div>
              </div>
            ) : (
              <div className="text-center font-sans animate-in slide-in-from-right-12">
                <h2 className="text-2xl font-serif text-[#1B365D] mb-4 text-center">Dados de Transferência</h2>
                <div className="bg-[#FAF9F8] p-8 rounded-[40px] border-2 border-[#1B365D]/5 mb-8 shadow-inner">
                  <QrCode size={100} className="mx-auto mb-6 text-[#1B365D]" strokeWidth={1} />
                  <p className="text-[9px] uppercase text-[#1B365D]/40 mb-3 font-black tracking-widest text-center">Chave Pix</p>
                  <div className="flex items-center justify-center space-x-2 bg-white px-5 py-3 border border-stone-100 rounded-full shadow-sm">
                    <span className="text-[11px] font-mono text-[#1B365D] truncate max-w-[150px]">{pixKey}</span>
                    <button onClick={copyPix} className="text-[#FACC15] hover:scale-125 transition-transform active:scale-90">
                      {copied ? <Check size={16} className="text-green-600" /> : <Copy size={16} />}
                    </button>
                  </div>
                </div>
                <button onClick={confirmReservation} className="w-full py-5 bg-[#1B365D] text-[#FACC15] text-[11px] uppercase tracking-[0.2em] font-black rounded-full shadow-xl hover:scale-[1.02] transition-all relative overflow-hidden pattern-blue">Já realizei o Pix</button>
                <button onClick={() => setPixStep(false)} className="mt-6 text-[10px] uppercase tracking-[0.3em] text-[#1B365D]/30 font-black hover:text-[#1B365D] text-center">Voltar</button>
              </div>
            )}
          </div>
        </div>
      )}

      {/* Rodapé com Linhas de Fundo */}
      <footer className="py-32 bg-[#1B365D] text-white text-center border-t-[12px] border-[#FACC15] relative overflow-hidden pattern-blue">
        <div className="relative z-10 max-w-lg mx-auto px-6 font-serif">
          <div className="flex justify-center items-center space-x-6 mb-12">
            <div className="h-[2px] w-12 bg-[#FACC15]/30"></div>
            <span className="text-5xl text-[#FACC15] font-light italic">D & E</span>
            <div className="h-[2px] w-12 bg-[#FACC15]/30"></div>
          </div>
          <p className="text-white/60 text-lg italic mb-12 leading-relaxed px-6 text-center">
            "Para nós, o amor é a única coisa que cresce quando se reparte."
          </p>
          <div className="inline-block bg-[#FACC15] px-12 py-4 rounded-full text-[14px] uppercase tracking-[0.6em] text-[#1B365D] font-black font-sans shadow-2xl">
            09.06.2026
          </div>
        </div>
      </footer>

      {/* Toast de Sucesso */}
      {showSuccess && (
        <div className="fixed bottom-12 left-1/2 -translate-x-1/2 z-50 bg-[#1B365D] text-[#FACC15] px-12 py-6 shadow-2xl flex items-center space-x-4 animate-in slide-in-from-bottom-20 rounded-full border-2 border-[#FACC15]/30">
          <Heart size={20} className="fill-[#FACC15] text-[#FACC15] animate-pulse" />
          <div className="flex flex-col text-left">
            <span className="text-[12px] uppercase tracking-[0.3em] font-black">Grazie!</span>
            <span className="text-[10px] text-white/60 uppercase tracking-widest font-sans">Daniel & Eduarda receberam o seu carinho.</span>
          </div>
        </div>
      )}

      <style dangerouslySetInnerHTML={{ __html: `
        @keyframes float {
          0%, 100% { transform: translateY(0) rotate(0deg); }
          50% { transform: translateY(-20px) rotate(10deg); }
        }
        .animate-float {
          animation: float 8s ease-in-out infinite;
        }
        .pattern-blue {
          background-image: repeating-linear-gradient(
            -45deg,
            rgba(255, 255, 255, 0.03),
            rgba(255, 255, 255, 0.03) 1px,
            transparent 1px,
            transparent 12px
          );
        }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        .line-clamp-3 {
          display: -webkit-box;
          -webkit-line-clamp: 3;
          -webkit-box-orient: vertical;
          overflow: hidden;
        }
      `}} />
    </div>
  );
}

```
