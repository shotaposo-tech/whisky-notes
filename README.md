import React, { useState, useEffect } from 'react';
import { db } from './firebase';
import { collection, addDoc, query, orderBy, onSnapshot, serverTimestamp } from 'firebase/firestore';
import MyRadarChart from './RadarChart'; // レーダーチャートをインポート

function App() {
  const [notes, setNotes] = useState([]);
  
  // フォームの入力状態を一括管理
  const [form, setForm] = useState({
    name: '',
    vintage: '',
    maltiness: 5,
    fruity: 5,
    smokiness: 2,
    oak: 4,
    palate: 5,
    finish: 5,
    guessDistillery: '', // ブラインド予想用
    comment: '',
  });

  // リアルタイムデータ取得
  useEffect(() => {
    const q = query(collection(db, "whiskyNotes"), orderBy("createdAt", "desc"));
    const unsubscribe = onSnapshot(q, (snapshot) => {
      setNotes(snapshot.docs.map(doc => ({ ...doc.data(), id: doc.id })));
    });
    return () => unsubscribe();
  }, []);

  // フォームの変更を処理
  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: (e.target.type === 'range' ? Number(value) : value) }));
  };

  // データを保存
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!form.name) return;

    await addDoc(collection(db, "whiskyNotes"), {
      ...form,
      createdAt: serverTimestamp(),
    });

    // フォームをリセット
    setForm({
      name: '', vintage: '', maltiness: 5, fruity: 5, smokiness: 2, oak: 4, palate: 5, finish: 5, guessDistillery: '', comment: '',
    });
  };

  return (
    <div style={{ maxWidth: '800px', margin: '0 auto', padding: '20px', fontFamily: '"Helvetica Neue", Arial, sans-serif', color: '#333', backgroundColor: '#fff' }}>
      <header style={{ textAlign: 'center', marginBottom: '40px' }}>
        <h1 style={{ fontSize: '2.5em', margin: '0', color: '#5d4037' }}>🥃 Whisky Tasting Lab</h1>
        <p style={{ color: '#8d6e63' }}>麦感を極める、一杯との対話。</p>
      </header>
      
      {/* 入力フォーム */}
      <form onSubmit={handleSubmit} style={{ background: '#fafafa', padding: '30px', borderRadius: '12px', boxShadow: '0 4px 6px rgba(0,0,0,0.05)', marginBottom: '40px' }}>
        <h2 style={{ borderBottom: '2px solid #a1887f', paddingBottom: '10px' }}>New Note</h2>
        
        {/* 基本情報 */}
        <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '15px', marginBottom: '20px' }}>
          <input type="text" name="name" value={form.name} onChange={handleChange} placeholder="銘柄名 (例: Ben Nevis 1996)" style={inputStyle} required />
          <input type="text" name="vintage" value={form.vintage} onChange={handleChange} placeholder="ヴィンテージ/熟成年数 (例: 1996 24yo)" style={inputStyle} />
        </div>

        {/* 味のプロファイル (スライダー) */}
        <div style={{ marginBottom: '20px' }}>
          <h3 style={{ fontSize: '1.1em', color: '#5d4037' }}>Flavor Profile (1-10)</h3>
          <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fit, minmax(200px, 1fr))', gap: '15px' }}>
            {[
              { id: 'maltiness', label: '麦感' }, { id: 'fruity', label: 'フルーティー' }, { id: 'smokiness', label: 'スモーキー' },
              { id: 'oak', label: 'オーク' }, { id: 'palate', label: '甘み・ボディ' }, { id: 'finish', label: '余韻' }
            ].map(flavor => (
              <div key={flavor.id} style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
                <label style={{ width: '100px', fontSize: '0.9em' }}>{flavor.label}: <span style={{ fontWeight: 'bold' }}>{form[flavor.id]}</span></label>
                <input type="range" name={flavor.id} min="1" max="10" value={form[flavor.id]} onChange={handleChange} style={{ flex: 1, accentColor: '#a1887f' }} />
              </div>
            ))}
          </div>
        </div>

        {/* テイスティングコメントとブラインド予想 */}
        <textarea name="comment" value={form.comment} onChange={handleChange} placeholder="コメント（香り、味わい、余韻などのニュアンス）" style={{ ...inputStyle, width: '100%', height: '100px', marginBottom: '15px' }} />
        
        <input type="text" name="guessDistillery" value={form.guessDistillery} onChange={handleChange} placeholder="ブラインド予想 (蒸留所/ボトラーなど)" style={{ ...inputStyle, width: '100%', marginBottom: '20px', backgroundColor: '#fffde7' }} />

        <button type="submit" style={buttonStyle}>ノートを保存</button>
      </form>

      {/* ノート一覧 */}
      <div>
        <h2 style={{ marginBottom: '20px', color: '#5d4037' }}>Notes Archive</h2>
        <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fit, minmax(350px, 1fr))', gap: '20px' }}>
          {notes.map(note => (
            <div key={note.id} style={noteCardStyle}>
              <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'flex-start', marginBottom: '15px' }}>
                <div>
                  <h3 style={{ margin: '0 0 5px 0', fontSize: '1.4em', color: '#3e2723' }}>{note.name}</h3>
                  <p style={{ margin: '0', color: '#8d6e63', fontSize: '0.9em' }}>{note.vintage || 'Vintage Unknown'}</p>
                </div>
                {note.createdAt && (
                  <span style={{ fontSize: '0.8em', color: '#bdbdbd' }}>
                    {note.createdAt.toDate().toLocaleDateString('ja-JP')}
                  </span>
                )}
              </div>

              {/* レーダーチャート */}
              <MyRadarChart data={note} />

              <p style={{ margin: '15px 0 0 0', lineHeight: '1.6', fontSize: '0.95em' }}>{note.comment}</p>
              
              {/* ブラインド予想の表示 (入力がある場合のみ) */}
              {note.guessDistillery && (
                <div style={guessStyle}>
                  <strong>予想:</strong> {note.guessDistillery}
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// 共通スタイル定義
const inputStyle = { padding: '12px', border: '1px solid #ddd', borderRadius: '6px', fontSize: '1em', outline: 'none' };
const buttonStyle = { width: '100%', padding: '15px', background: '#5d4037', color: '#fff', border: 'none', borderRadius: '8px', cursor: 'pointer', fontSize: '1.1em', fontWeight: 'bold' };
const noteCardStyle = { background: '#fafafa', border: '1px solid #eee', padding: '20px', borderRadius: '12px', transition: 'all 0.3s ease' };
const guessStyle = { marginTop: '15px', padding: '10px', background: '#fffde7', borderRadius: '6px', fontSize: '0.9em', color: '#616161', border: '1px solid #fff9c4' };

export default App;
