const SUPABASE_URL = 'https://abkkjbbdqbieiivdbxad.supabase.co';
const SUPABASE_KEY = 'sb_publishable_vjGeUvVBveqlD_ipO6y5Bw_EUE-z_TX';

const recipeSchema = {
  type: 'object',
  additionalProperties: false,
  required: ['title','emoji','meal_type','servings','prep_minutes','cook_minutes','calories','protein_g','carbs_g','fat_g','ingredients','method','notes'],
  properties: {
    title: {type:'string'},
    emoji: {type:'string'},
    meal_type: {type:'string', enum:['breakfast','lunch','tea','snack']},
    servings: {type:'integer', minimum:1, maximum:30},
    prep_minutes: {type:'integer', minimum:0, maximum:1440},
    cook_minutes: {type:'integer', minimum:0, maximum:1440},
    calories: {type:'integer', minimum:0, maximum:10000},
    protein_g: {type:'number', minimum:0, maximum:1000},
    carbs_g: {type:'number', minimum:0, maximum:2000},
    fat_g: {type:'number', minimum:0, maximum:1000},
    ingredients: {
      type:'array', minItems:1, maxItems:80,
      items:{
        type:'object', additionalProperties:false,
        required:['raw_text','name','quantity','unit','category'],
        properties:{
          raw_text:{type:'string'},
          name:{type:'string'},
          quantity:{type:'number', minimum:0, maximum:100000},
          unit:{type:'string'},
          category:{type:'string', enum:['Meat','Fish','Fruit & Veg','Dairy','Bakery','Cupboard','Frozen','Drinks','Household','Other']}
        }
      }
    },
    method:{type:'array', minItems:1, maxItems:50, items:{type:'string'}},
    notes:{type:'string'}
  }
};

function json(res,status,obj){
  res.statusCode=status;
  res.setHeader('Content-Type','application/json; charset=utf-8');
  res.setHeader('Cache-Control','no-store');
  res.end(JSON.stringify(obj));
}

async function verifyUser(req){
  const auth=req.headers.authorization||'';
  if(!auth.startsWith('Bearer ')) return null;
  const r=await fetch(`${SUPABASE_URL}/auth/v1/user`,{headers:{apikey:SUPABASE_KEY,Authorization:auth}});
  if(!r.ok) return null;
  return r.json();
}

function safeUrl(value){
  let u;
  try{u=new URL(value)}catch{return null}
  if(!['http:','https:'].includes(u.protocol)) return null;
  const h=u.hostname.toLowerCase();
  if(h==='localhost'||h.endsWith('.local')||h==='0.0.0.0'||h==='127.0.0.1'||h==='::1') return null;
  if(/^10\.|^192\.168\.|^169\.254\.|^172\.(1[6-9]|2\d|3[01])\./.test(h)) return null;
  return u;
}

function htmlToText(html){
  return html
    .replace(/<script\b[^>]*>[\s\S]*?<\/script>/gi,' ')
    .replace(/<style\b[^>]*>[\s\S]*?<\/style>/gi,' ')
    .replace(/<svg\b[^>]*>[\s\S]*?<\/svg>/gi,' ')
    .replace(/<[^>]+>/g,' ')
    .replace(/&nbsp;/gi,' ')
    .replace(/&amp;/gi,'&')
    .replace(/&quot;/gi,'"')
    .replace(/&#39;/gi,"'")
    .replace(/\s+/g,' ')
    .trim();
}

async function fetchRecipePage(url){
  const u=safeUrl(url);
  if(!u) throw new Error('That link is not allowed.');
  const ctrl=new AbortController();
  const timer=setTimeout(()=>ctrl.abort(),8000);
  try{
    const r=await fetch(u,{redirect:'follow',signal:ctrl.signal,headers:{'User-Agent':'TeaPlanRecipeImporter/1.0','Accept':'text/html,application/xhtml+xml'}});
    if(!r.ok) throw new Error(`Could not read that page (${r.status}).`);
    const type=r.headers.get('content-type')||'';
    if(!type.includes('text/html')&&!type.includes('text/plain')) throw new Error('That link is not a readable recipe page.');
    const html=(await r.text()).slice(0,1_500_000);
    const text=htmlToText(html).slice(0,45_000);
    if(text.length<120) throw new Error('Not enough recipe text was found on that page. Try a screenshot instead.');
    return text;
  }finally{clearTimeout(timer)}
}

module.exports = async function handler(req,res){
  if(req.method!=='POST') return json(res,405,{error:'Method not allowed'});
  if(!process.env.OPENAI_API_KEY) return json(res,500,{error:'TeaPlan AI is not configured yet.'});
  const user=await verifyUser(req);
  if(!user) return json(res,401,{error:'Please sign in to TeaPlan again.'});

  let body=req.body;
  if(typeof body==='string'){
    try{body=JSON.parse(body)}catch{return json(res,400,{error:'Invalid request.'})}
  }
  body=body||{};
  const mode=String(body.mode||'text');
  let sourceText='';
  let sourceLabel='';
  const content=[];

  try{
    if(mode==='url'){
      const url=String(body.url||'').trim();
      if(!url) return json(res,400,{error:'Paste a recipe link first.'});
      sourceText=await fetchRecipePage(url);
      sourceLabel=`Recipe webpage: ${url}`;
    }else if(mode==='image'){
      const image=String(body.image||'');
      if(!/^data:image\/(jpeg|jpg|png|webp);base64,/i.test(image)) return json(res,400,{error:'Please choose a JPG, PNG or WEBP image.'});
      if(image.length>7_000_000) return json(res,413,{error:'That image is too large. Please choose a smaller screenshot/photo.'});
      sourceLabel='Photo or screenshot supplied by the user.';
      content.push({type:'input_image',image_url:image,detail:'high'});
    }else{
      sourceText=String(body.text||'').trim().slice(0,45_000);
      if(!sourceText) return json(res,400,{error:'Paste some recipe text first.'});
      sourceLabel='Recipe text supplied by the user.';
    }

    const instructions=`You extract cooking recipes for TeaPlan, a UK household meal-planning app. Return one clean recipe. Preserve ingredient quantities and units accurately. Normalise ingredient names for shopping-list merging, but keep the original wording in raw_text. Use UK-friendly units where the source is unambiguous; do not invent conversions when uncertain. Suggest a meal as breakfast, lunch, tea, or snack, but this is only a default suggestion because TeaPlan lets users place any recipe in any weekly meal slot. Estimate calories, protein, carbs and fat PER SERVING from the listed ingredient quantities; these are estimates and must be plausible. If nutrition is explicitly provided, prefer it. prep_minutes is hands-on preparation time and cook_minutes is cooking time. For ingredients such as salt 'to taste' where no numeric amount exists, set quantity to 0 and unit to an empty string while retaining the wording. Method must be a clear ordered list. Choose a single relevant food emoji. Do not include commentary outside the JSON.`;

    content.unshift({type:'input_text',text:`${sourceLabel}\n\n${sourceText ? 'SOURCE CONTENT:\n'+sourceText : 'Read the attached recipe image carefully.'}`});

    const payload={
      model:'gpt-5.4-mini',
      instructions,
      input:[{role:'user',content}],
      text:{format:{type:'json_schema',name:'teaplan_recipe',description:'A structured recipe for TeaPlan',strict:true,schema:recipeSchema}}
    };

    const ai=await fetch('https://api.openai.com/v1/responses',{
      method:'POST',
      headers:{'Authorization':`Bearer ${process.env.OPENAI_API_KEY}`,'Content-Type':'application/json'},
      body:JSON.stringify(payload)
    });
    const result=await ai.json();
    if(!ai.ok){
      const msg=result?.error?.message||'OpenAI could not process that recipe.';
      return json(res,502,{error:msg});
    }
    const outputText=result.output_text || result.output?.flatMap(x=>x.content||[]).find(x=>x.type==='output_text')?.text;
    if(!outputText) return json(res,502,{error:'TeaPlan AI returned no recipe.'});
    let recipe;
    try{recipe=JSON.parse(outputText)}catch{return json(res,502,{error:'TeaPlan AI returned an unreadable recipe.'})}
    return json(res,200,{recipe});
  }catch(e){
    const message=e?.name==='AbortError'?'That recipe page took too long to respond. Try a screenshot instead.':(e?.message||'Could not import that recipe.');
    return json(res,400,{error:message});
  }
}
