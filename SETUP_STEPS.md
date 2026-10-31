# LexiNode.ai setup (new account, ~45 min)

## 0. Instagram account (5 min)
1. Account **Professional** karaa: Settings and activity -> Account type and tools -> Switch to professional account -> **Creator** (kinva Business).
2. Profile picture: `assets/profile_picture.png`.
3. Name: `LexiNode.ai | Code DSA AI`  (Name field search madhe disto)
4. Bio (copy-paste):
   ```
   Interview-ready tech in 60 seconds
   System Design | DSA | AI | Automation
   Save this for your next interview 🔖
   ```
5. Highlights: 4 covers `assets/cover_*.png` (pahile 4 Reels nantar highlights banva).

## 1. Meta app madhe navin account add kara (10 min)
Junya "Mind Meets Machine" sathi banvlela **ch Meta app vapra** (navin app lagat nahi).
1. developers.facebook.com -> My Apps -> tujha app -> **App roles -> Roles -> Add People -> Instagram Testers** -> `lexinode.ai` username taka.
2. Instagram app madhe (lexinode.ai account): Settings -> **Website permissions -> Apps and websites -> Tester invites** -> Accept.
3. Meta app -> Products -> **Instagram -> API setup with Instagram login -> Generate access token** (lexinode.ai chya samor "Add account" -> login -> Allow).
4. Don goshti copy kara: **Instagram account ID** (IG_USER_ID) ani **Access token** (IG_ACCESS_TOKEN). Token koni la dakhvu naka.
   (Permissions: instagram_business_basic ani instagram_business_content_publish.)

## 2. GitHub: navin repo (10 min)
1. github.com -> New repository -> name `lexinode-automation` -> **Private** -> Create.
2. Ya folder madhle sarv files upload kara (**.github/workflows** folder sudha: "Add file -> Upload files", folder drag karun).
3. Settings -> Secrets and variables -> Actions -> **Secrets** (New repository secret):
   | Name | Value |
   |---|---|
   | GEMINI_API_KEY | junhach key chalel (aistudio.google.com/apikey) |
   | GROQ_API_KEY | junhach (optional pan recommended) |
   | IG_USER_ID | lexinode.ai cha account ID |
   | IG_ACCESS_TOKEN | lexinode.ai cha token |
   | CLOUDINARY_CLOUD_NAME | junhach |
   | CLOUDINARY_UPLOAD_PRESET | junhach (unsigned preset) |
   | GH_PAT | GitHub token (Settings -> Developer settings -> Fine-grained token, **ya repo** sathi, permission: Secrets = Read and write). Token weekly refresh sathi. |
4. **Variables** tab (optional): `IG_HANDLE` = `lexinode.ai`, `BRAND_NAME` = `LexiNode.ai`, `BRAND_MARK` = `LN`, `EXPLAIN_VOICE` = `hi-IN-MadhurNeural` (kinva `hi-IN-SwaraNeural`).

## 3. Test (10 min)
1. Actions -> **LexiNode explainer Reel** -> Run workflow -> `diagnose` = true -> Run. Log madhe account username ani quota dista pahije.
2. Mag Run workflow -> `dry_run` = true (default) -> Run. Shevti **Artifacts -> lexinode-output** download kara: video baghha, voice aika.
3. Chhan watla tar Run workflow -> `dry_run` = **false** -> pahila Reel post hoto.
4. Aata schedule apoap: roj **19:10 IST**. Dusra post (09:00 IST) suru karayla Variables madhe `SECOND_POST` = `on` (1-2 athavde nantar).

## Navin account sathi niyam
- Pahile 1-2 athavde roj 1 Reel. Navin account var jast posting automation la spam vatu shakte.
- Instagram app madhun stha 3-5 posts manually kara, 20-30 relevant accounts follow kara (bulk follow/like karu naka).
- Followers cha guarantee nahi. 5-6 divsa nantar Insights baghha: saves ani watch time mahatvache.
- Actions free minutes: private repo la ~2000 min/mahina. Roj 1-2 Reels = ~250 min. Junya repo cha usage Settings -> Billing madhe baghha.
