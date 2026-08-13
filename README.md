import streamlit as st
from openai import OpenAI
import requests
import json
import re

st.set_page_config(page_title="農村歌謡ブログ生成＆全自動投稿", page_icon="🌾")

st.title("🌾 農村歌謡ブログ生成＆全自動投稿システム")
st.caption("YouTube URLを貼るだけで！歌詞抽出・記事生成・WordPress下書き保存まで完結します。")

# サイドバー設定
with st.sidebar:
    st.header("⚙️ API・接続設定")
    openai_api_key = st.text_input("OpenAI API Key", type="password")
    wp_url = st.text_input("WordPress URL", placeholder="https://your-site.com")
    wp_user = st.text_input("WordPress ユーザー名", value="ecobon")
    wp_app_pass = st.text_input("WordPress アプリケーションパスワード", type="password")

youtube_url = st.text_input("YouTube動画のURLを入力してください")

def extract_video_id(url):
    pattern = r'(?:v=|\/)([0-9A-Za-z_-]{11})'
    match = re.search(pattern, url)
    return match.group(1) if match else None

if st.button("🚀 ワンクリックで全自動生成＆保存", type="primary"):
    if not (openai_api_key and wp_url and wp_user and wp_app_pass and youtube_url):
        st.error("設定項目とYouTube URLをすべて入力してください。")
    else:
        video_id = extract_video_id(youtube_url)
        if not video_id:
            st.error("有効なYouTube URLではありません。")
        else:
            try:
                st.info("🔄 記事を生成中...")
                client = OpenAI(api_key=openai_api_key)
                
                # ChatGPTによる記事生成プロンプト
                prompt = f"""
                YouTube動画(ID: {video_id})のテーマ・歌詞を元に、農村歌謡ブログ記事を作成してください。
                
                【構成案】
                ・魅力的でSEOに適したタイトル
                ・導入文
                ・見出し（h2, h3）を用いた本文（農村文化、自然循環、農業の知恵などの視点を含める）
                ・まとめ
                ・おすすめタグ（カンマ区切り）
                
                HTMLタグ（<h2>, <p>など）を含めて出力してください。
                """
                
                response = client.chat.completions.create(
                    model="gpt-4o-mini",
                    messages=[{"role": "user", "content": prompt}]
                )
                
                content = response.choices[0].message.content
                st.success("✨ 記事の生成が完了しました！")
                st.markdown(content, unsafe_allow_html=True)
                
                # WordPress投稿処理
                st.info("🚀 WordPressへ投稿中...")
                clean_wp_url = wp_url.rstrip('/')
                api_endpoint = f"{clean_wp_url}/wp-json/wp/v2/posts"
                
                post_data = {
                    "title": f"農村歌謡ブログ：動画解説 ({video_id})",
                    "content": content,
                    "status": "draft"
                }
                
                res = requests.post(
                    api_endpoint,
                    auth=(wp_user, wp_app_pass),
                    json=post_data
                )
                
                if res.status_code == 201:
                    st.balloons()
                    st.success("🎉 WordPressへ「下書き」として正常に保存されました！")
                else:
                    st.error(f"WordPress投稿失敗 ({res.status_code}): {res.text}")
                    
            except Exception as e:
                st.error(f"エラーが発生しました: {str(e)}")
