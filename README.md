/**
 * MailPro.ci — Formulaires de commande (Basic / Premium / Platinium) + Newsletter
 * ===================================================================
 * FRANÇAIS par défaut. ANGLAIS quand l'adresse contient &lang=en (ex : ...exec?pack=Basic&lang=en).
 * La page d'accueil ajoute déjà « lang=en » aux boutons quand le visiteur choisit EN.
 *
 * 1. doGet()  -> affiche le formulaire (bouton "Choisir Basic", etc.)
 * 2. soumettreCommande() -> enregistre la demande dans le Google Sheet (un onglet par pack),
 *                te prévient par e-mail, puis donne AU CLIENT son lien de paiement :
 *                affiché à l'écran ET envoyé par e-mail, dans sa langue.
 *
 * Après toute modification de ce fichier : Déployer > Gérer les déploiements >
 * icône crayon > Version : "Nouvelle version" > Déployer. (L'URL ne change pas.)
 *
 * ⚠️ Le lien de paiement ne confirme pas que le client a payé : vérifie toujours
 * le paiement dans ta banque avant de configurer l'adresse e-mail.
 */

const PACKS = {
  Basic: {
    nom: "Basic", collaborateurs: false,
    prix: "18 000 FCFA / an", prixCourt: "18 000 FCFA",
    prixEn: "18,000 FCFA / year (≈ €27)", prixCourtEn: "18,000 FCFA",
    lien: "https://ebank.gtbankci.com/MerchandQRCode/Home/PayMe?pay2=9ba20148-66d9-4873-9772-52dd83cb6ae2"
  },
  Premium: {
    nom: "Premium", collaborateurs: false,
    prix: "35 000 FCFA / an", prixCourt: "35 000 FCFA",
    prixEn: "35,000 FCFA / year (≈ €53)", prixCourtEn: "35,000 FCFA",
    lien: "https://ebank.gtbankci.com/MerchandQRCode/Home/PayMe?pay2=71db0917-20e1-4729-9853-9edb89de0850"
  },
  // sur devis : pas de lien de paiement
  Platinium: { nom: "Platinium", collaborateurs: true, prix: "Sur devis", prixCourt: "Sur devis", prixEn: "On quote", prixCourtEn: "On quote", lien: "" }
};

// Adresse qui reçoit une notification à chaque nouvelle commande.
const EMAIL_NOTIFICATION = "support@mailpro.ci";
const WHATSAPP = "https://wa.me/2250708519651";

// ===================== Langues =====================
function langue_(v) { return String(v || "").toLowerCase().indexOf("en") === 0 ? "en" : "fr"; }

const L_ = {
  fr: {
    invalidTitre: "Lien invalide",
    invalidCorps: "<p>Ce lien de commande n'est pas reconnu. Merci de revenir sur <a href='https://www.mailpro.ci'>mailpro.ci</a> et de réessayer.</p>",
    packInvalide: "Pack invalide.",
    erreurGenerale: "Une erreur est survenue. Réessayez ou écrivez-nous sur WhatsApp.",
    recuTitre: "Demande reçue ✅", derniere: "Merci ! Dernière étape : payez votre pack",
    payer: function (c) { return "Payer " + c + " maintenant"; },
    noteLien: "Nous vous avons aussi envoyé ce lien par e-mail. Une fois le paiement fait, notre équipe vous contacte sur WhatsApp sous 30 minutes pour configurer votre adresse.",
    question: "Une question ? WhatsApp",
    devisTitre: "Demande envoyée !",
    devisIntro: function (nom) { return "<p>Merci ! Votre demande pour le pack <strong>" + nom + "</strong> a bien été reçue.</p>"; },
    devisSuite: "Notre équipe vous recontacte très vite pour établir votre devis.", nousEcrire: "Nous écrire sur WhatsApp",
    // formulaire
    badge: function (nom, prix) { return "Pack " + nom + " · " + prix; },
    h1: function (nom) { return "Votre e-mail <span class='accent'>" + nom + "</span> commence ici"; },
    subLien: "Remplissez le formulaire : vous recevez tout de suite votre lien de paiement, à l'écran et par e-mail.",
    subDevis: "Remplissez le formulaire, notre équipe vous recontacte très vite pour établir votre devis.",
    entreprise: "Nom de l'entreprise / marque", nom: "Votre nom", tel: "Téléphone / WhatsApp", emailActuel: "Votre e-mail actuel",
    emailPro: "E-mail professionnel souhaité", collab: "Nombre de collaborateurs", collabEx: "Ex : 15",
    envoyer: "Envoyer ma demande", envoi: "Envoi en cours...", foot: "Paiement sécurisé · Accès sous 30 min après paiement",
    // e-mail client
    mailSujet: function (nom) { return "Votre demande MailPro.ci — Pack " + nom; },
    mailIntro: function (nom, pack, prix) { return "Bonjour " + (nom || "") + ",\n\nMerci ! Nous avons bien reçu votre demande pour le pack " + pack + " (" + prix + ").\n\n"; },
    mailLien: function (lien, ep) { return "Pour finaliser, payez avec ce lien sécurisé :\n" + lien + "\n\nUne fois le paiement effectué, notre équipe vous contacte sur WhatsApp sous 30 minutes pour configurer votre adresse " + (ep || "") + ".\n\n"; },
    mailDevis: "Notre équipe vous recontacte très vite pour établir votre devis.\n\n",
    mailFin: function (wa) { return "Une question ? Écrivez-nous sur WhatsApp : " + wa + "\n\nL'équipe MailPro.ci"; },
    // newsletter
    nlTitre: "Guide gratuit", nlBadge: "Guide gratuit",
    nlH1: "Les 5 erreurs à éviter avec votre <span class='accent'>e-mail professionnel</span>",
    nlSub: "Recevez le guide en PDF dans votre boîte mail, puis un conseil par mois. Désinscription en 1 clic.",
    nlPrenom: "Votre prénom", nlEmail: "Votre e-mail", nlBouton: "Recevoir mon guide gratuit",
    nlFoot: "En vous inscrivant, vous acceptez de recevoir nos e-mails. Aucune revente de données.",
    nlGuideTitre: "5 erreurs à éviter avec votre e-mail professionnel",
    nlEmailInvalide: "Adresse e-mail invalide.",
    nlIndispo: "Le guide est momentanément indisponible. Écrivez-nous sur WhatsApp, nous vous l'envoyons.",
    nlDejaTitre: "Déjà inscrit(e) ✅",
    nlDeja: function (e) { return "<p>L'adresse <strong>" + e + "</strong> est déjà inscrite : le guide vous a été envoyé. Pensez à vérifier vos spams, ou écrivez-nous sur WhatsApp.</p>"; },
    nlPlafond: "Beaucoup d'inscriptions aujourd'hui : réessayez demain ou écrivez-nous sur WhatsApp pour recevoir le guide.",
    nlMailSujet: function (t) { return "Votre guide gratuit : " + t; },
    nlMailCorps: function (nom, t, site, wa, desabo) { return "Bonjour " + nom + ",\n\nMerci de votre inscription ! Voici votre guide « " + t + " », en pièce jointe (PDF).\n\n" +
      "Besoin d'une adresse e-mail professionnelle ? Découvrez nos offres : " + site + "\n" + "Une question ? Écrivez-nous sur WhatsApp : " + wa + "\n\nL'équipe MailPro.ci\n\n" +
      "—\nVous recevez cet e-mail car vous vous êtes inscrit(e) sur mailpro.ci.\nPour vous désinscrire en 1 clic : " + desabo; },
    nlOkTitre: "Guide envoyé ✅",
    nlOk: function (t, e) { return "<p>Votre guide <strong>« " + t + " »</strong> vient d'être envoyé à <strong>" + e + "</strong>.</p><p class='note'>Pensez à vérifier vos spams si vous ne le voyez pas dans quelques minutes.</p>"; },
    nlOffres: "Découvrir nos offres", merci: "Merci !",
    desaboOkTitre: "Vous êtes désinscrit(e)", desaboOk: function (e) { return "<p>L'adresse <strong>" + e + "</strong> ne recevra plus nos e-mails. Merci !</p>"; },
    desaboKoTitre: "Lien invalide", desaboKo: "<p>Ce lien de désinscription n'est pas valide. Écrivez-nous à support@mailpro.ci.</p>"
  },
  en: {
    invalidTitre: "Invalid link",
    invalidCorps: "<p>This order link is not recognised. Please go back to <a href='https://www.mailpro.ci'>mailpro.ci</a> and try again.</p>",
    packInvalide: "Invalid pack.",
    erreurGenerale: "Something went wrong. Please try again or message us on WhatsApp.",
    recuTitre: "Request received ✅", derniere: "Thank you! Last step: pay for your",
    payer: function (c) { return "Pay " + c + " now"; },
    noteLien: "We also sent you this link by e-mail. Once your payment is made, our team will contact you on WhatsApp within 30 minutes to set up your address.",
    question: "Any question? WhatsApp",
    devisTitre: "Request sent!",
    devisIntro: function (nom) { return "<p>Thank you! Your request for the <strong>" + nom + "</strong> pack has been received.</p>"; },
    devisSuite: "Our team will get back to you shortly with a quote.", nousEcrire: "Message us on WhatsApp",
    badge: function (nom, prix) { return nom + " pack · " + prix; },
    h1: function (nom) { return "Your <span class='accent'>" + nom + "</span> e-mail starts here"; },
    subLien: "Fill in the form: you will get your payment link right away, on screen and by e-mail.",
    subDevis: "Fill in the form and our team will get back to you shortly with a quote.",
    entreprise: "Company / brand name", nom: "Your name", tel: "Phone / WhatsApp", emailActuel: "Your current e-mail",
    emailPro: "Desired professional e-mail", collab: "Number of employees", collabEx: "e.g. 15",
    envoyer: "Send my request", envoi: "Sending...", foot: "Secure payment · Access within 30 min after payment",
    mailSujet: function (nom) { return "Your MailPro.ci request — " + nom + " pack"; },
    mailIntro: function (nom, pack, prix) { return "Hello " + (nom || "") + ",\n\nThank you! We have received your request for the " + pack + " pack (" + prix + ").\n\n"; },
    mailLien: function (lien, ep) { return "To finalize, pay with this secure link:\n" + lien + "\n\nOnce your payment is made, our team will contact you on WhatsApp within 30 minutes to set up your address " + (ep || "") + ".\n\n"; },
    mailDevis: "Our team will get back to you shortly to prepare your quote.\n\n",
    mailFin: function (wa) { return "Any question? Message us on WhatsApp: " + wa + "\n\nThe MailPro.ci team"; },
    nlTitre: "Free guide", nlBadge: "Free guide",
    nlH1: "The 5 mistakes to avoid with your <span class='accent'>professional e-mail</span>",
    nlSub: "Get the guide as a PDF in your inbox, then one tip a month. Unsubscribe in 1 click.",
    nlPrenom: "Your first name", nlEmail: "Your e-mail", nlBouton: "Get my free guide",
    nlFoot: "By signing up, you agree to receive our e-mails. We never sell your data.",
    nlGuideTitre: "The 5 mistakes to avoid with your professional e-mail",
    nlEmailInvalide: "Invalid e-mail address.",
    nlIndispo: "The guide is temporarily unavailable. Message us on WhatsApp and we will send it to you.",
    nlDejaTitre: "Already subscribed ✅",
    nlDeja: function (e) { return "<p>The address <strong>" + e + "</strong> is already subscribed: the guide was sent to you. Please check your spam folder, or message us on WhatsApp.</p>"; },
    nlPlafond: "Lots of sign-ups today: please try again tomorrow, or message us on WhatsApp to receive the guide.",
    nlMailSujet: function (t) { return "Your free guide: " + t; },
    nlMailCorps: function (nom, t, site, wa, desabo, fr) { return "Hello " + nom + ",\n\nThank you for signing up! Here is your guide \"" + t + "\", attached (PDF).\n" +
      (fr ? "(The guide is currently available in French only; an English edition is coming soon.)\n" : "") + "\n" +
      "Need a professional e-mail address? See our offers: " + site + "\n" + "Any question? Message us on WhatsApp: " + wa + "\n\nThe MailPro.ci team\n\n" +
      "—\nYou are receiving this e-mail because you signed up on mailpro.ci.\nTo unsubscribe in 1 click: " + desabo; },
    nlOkTitre: "Guide sent ✅",
    nlOk: function (t, e) { return "<p>Your guide <strong>\"" + t + "\"</strong> has just been sent to <strong>" + e + "</strong>.</p><p class='note'>Please check your spam folder if you don't see it within a few minutes.</p>"; },
    nlOffres: "See our offers", merci: "Thank you!",
    desaboOkTitre: "You are unsubscribed", desaboOk: function (e) { return "<p>The address <strong>" + e + "</strong> will no longer receive our e-mails. Thank you!</p>"; },
    desaboKoTitre: "Invalid link", desaboKo: "<p>This unsubscribe link is not valid. Please write to support@mailpro.ci.</p>"
  }
};

function prixAffiche_(info, lang) { return lang === "en" ? info.prixEn : info.prix; }
function prixCourt_(info, lang) { return lang === "en" ? info.prixCourtEn : info.prixCourt; }

function doGet(e) {
  const lang = langue_(e.parameter.lang);
  const L = L_[lang];

  // Newsletter : désinscription et formulaire du guide gratuit
  if (e.parameter.action === "desabo") return desinscription_(e);
  if (e.parameter.pack === "Newsletter") {
    return HtmlService.createHtmlOutput(newsletterHtml_(e.parameter.source || "", lang))
      .setTitle("MailPro.ci — " + L.nlTitre)
      .addMetaTag("viewport", "width=device-width, initial-scale=1");
  }

  const pack = e.parameter.pack;
  const info = PACKS[pack];
  const actionUrl = ScriptApp.getService().getUrl();

  if (!info) {
    return HtmlService.createHtmlOutput(page(L.invalidTitre, L.invalidCorps));
  }

  return HtmlService.createHtmlOutput(formHtml(pack, info, actionUrl, lang))
    .setTitle("MailPro.ci — " + info.nom)
    .addMetaTag("viewport", "width=device-width, initial-scale=1");
}

// Évite qu'un texte commençant par = + - @ soit interprété comme une formule dans le Sheet
function sur_(valeur) {
  const v = String(valeur || "");
  return /^[=+\-@]/.test(v) ? "'" + v : v;
}

// Contenu de la page de confirmation (titre + corps). Seules des valeurs du script y apparaissent.
function confirmationHtml_(info, lang) {
  const L = L_[lang || "fr"];
  if (info.lien) {
    return {
      titre: L.recuTitre,
      corps: "<p>" + L.derniere + " <strong>" + info.nom + "</strong> (" + prixAffiche_(info, lang) + ").</p>" +
        "<a class='btn' href='" + info.lien + "' target='_blank' rel='noopener'>" + L.payer(prixCourt_(info, lang)) + "</a>" +
        "<p class='note'>" + L.noteLien + "</p>" +
        "<a class='btn2' href='" + WHATSAPP + "' target='_blank' rel='noopener'>" + L.question + "</a>"
    };
  }
  return {
    titre: L.devisTitre,
    corps: L.devisIntro(info.nom) + "<p>" + L.devisSuite + "</p>" +
      "<a class='btn' href='" + WHATSAPP + "' target='_blank' rel='noopener'>" + L.nousEcrire + "</a>"
  };
}

// Numéro de la colonne « Langue » (créée à la fin de la ligne d'en-têtes si elle n'existe pas)
function colonneLangue_(sheet) {
  const nb = Math.max(sheet.getLastColumn(), 1);
  const entetes = sheet.getRange(1, 1, 1, nb).getValues()[0];
  for (let i = 0; i < entetes.length; i++) {
    if (String(entetes[i]).toLowerCase().indexOf("langue") !== -1) return i + 1;
  }
  sheet.getRange(1, nb + 1).setValue("Langue");
  return nb + 1;
}

// Enregistre la commande, prévient par e-mail, et retourne la page de confirmation à afficher
function enregistrerCommande_(p) {
  const lang = langue_(p.lang);
  const L = L_[lang];
  const pack = p.pack;
  const info = PACKS[pack];
  if (!info) return { ok: false, message: L.packInvalide };

  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(pack);
  if (!sheet) {
    sheet = ss.insertSheet(pack);
    const headers = ["Date", "Entreprise", "Nom du client", "Téléphone / WhatsApp", "Email actuel", "Email pro souhaité"];
    if (info.collaborateurs) headers.push("Nombre de collaborateurs");
    sheet.appendRow(headers);
  }

  const row = [new Date(), sur_(p.entreprise), sur_(p.nom), sur_(p.telephone), sur_(p.email_actuel), sur_(p.email_pro)];
  if (info.collaborateurs) row.push(sur_(p.collaborateurs));
  sheet.appendRow(row);
  // Langue du client (sert pour la facture) : colonne « Langue », FR ou EN
  sheet.getRange(sheet.getLastRow(), colonneLangue_(sheet)).setValue(lang.toUpperCase());

  // --- Notification par e-mail à toi (toujours en français) ---
  try {
    let detail = "Pack : " + info.nom + " (" + info.prix + ")\n" +
      "Langue du client : " + lang.toUpperCase() + "\n" +
      "Entreprise : " + (p.entreprise || "") + "\n" +
      "Nom du client : " + (p.nom || "") + "\n" +
      "Téléphone / WhatsApp : " + (p.telephone || "") + "\n" +
      "Email actuel : " + (p.email_actuel || "") + "\n" +
      "Email pro souhaité : " + (p.email_pro || "");
    if (info.collaborateurs) detail += "\nNombre de collaborateurs : " + (p.collaborateurs || "");
    detail += "\n\nLien de paiement donné au client : " + (info.lien ? "oui" : "non (pack sur devis)") +
      "\nÀ faire : vérifier le paiement dans la banque, puis configurer l'adresse.";
    MailApp.sendEmail(EMAIL_NOTIFICATION, "Nouvelle commande MailPro (" + lang.toUpperCase() + ") — Pack " + info.nom, detail);
  } catch (err) {}

  // --- E-mail au client, dans sa langue (avec le lien de paiement s'il y en a un) ---
  try {
    if (p.email_actuel) {
      let corps = L.mailIntro(p.nom, info.nom, prixAffiche_(info, lang));
      corps += info.lien ? L.mailLien(info.lien, p.email_pro) : L.mailDevis;
      corps += L.mailFin(WHATSAPP);
      MailApp.sendEmail(p.email_actuel, L.mailSujet(info.nom), corps);
    }
  } catch (err) {}

  const c = confirmationHtml_(info, lang);
  return { ok: true, titre: c.titre, corps: c.corps };
}

// Appelée par la page du formulaire (la confirmation s'affiche sans recharger la page)
function soumettreCommande(form) {
  try {
    return enregistrerCommande_(form || {});
  } catch (err) {
    return { ok: false, message: L_[langue_((form || {}).lang)].erreurGenerale };
  }
}

// Ancienne méthode (envoi classique du formulaire), gardée au cas où
function doPost(e) {
  const r = enregistrerCommande_(e.parameter);
  const html = r.ok ? page(r.titre, r.corps) : page("Erreur", "<p>" + r.message + "</p>");
  return HtmlService.createHtmlOutput(html).addMetaTag("viewport", "width=device-width, initial-scale=1");
}

// Script du formulaire : envoie la commande puis affiche la confirmation dans la même page.
// Les textes (envoi en cours, erreur...) viennent des attributs data-* du formulaire : ils suivent la langue.
const SCRIPT_FORM_ = `
function envoyer(ev) {
  ev.preventDefault();
  var f = document.getElementById('f');
  var b = f.querySelector('button');
  b.disabled = true;
  b.textContent = f.dataset.envoi;
  google.script.run
    .withSuccessHandler(function (r) {
      if (r && r.ok) {
        document.getElementById('carte').innerHTML = '<div style="text-align:center"><h1>' + r.titre + '</h1>' + r.corps + '</div>';
        window.scrollTo(0, 0);
      } else {
        b.disabled = false;
        b.textContent = f.dataset.bouton;
        alert((r && r.message) || f.dataset.erreur);
      }
    })
    .withFailureHandler(function () {
      b.disabled = false;
      b.textContent = f.dataset.bouton;
      alert(f.dataset.erreur);
    })
    .soumettreCommande(f);
  return false;
}
`;

function css_() {
  return "body{font-family:-apple-system,Segoe UI,Roboto,Helvetica,Arial,sans-serif;background:#F7F7FB;margin:0;padding:0;color:#0B1220;}" +
    ".wrap{max-width:480px;margin:0 auto;padding:32px 20px 60px;}" +
    ".badge{display:inline-block;background:#EDE9FE;color:#6D28D9;font-size:13px;font-weight:600;padding:6px 14px;border-radius:999px;margin-bottom:16px;}" +
    "h1{font-size:26px;line-height:1.25;margin:0 0 8px;}" +
    ".accent{background:linear-gradient(90deg,#4F46E5,#7C3AED);-webkit-background-clip:text;background-clip:text;color:transparent;}" +
    ".sub{font-size:15px;color:#6B7280;margin:0 0 28px;}" +
    "label{display:block;font-size:14px;font-weight:600;margin:18px 0 6px;}" +
    "input{width:100%;box-sizing:border-box;padding:13px 14px;border:1.5px solid #E5E7EB;border-radius:10px;font-size:15px;font-family:inherit;}" +
    "input:focus{outline:none;border-color:#7C3AED;}" +
    ".btn{display:block;width:100%;box-sizing:border-box;text-align:center;background:#10B981;color:#fff;border:none;padding:15px;border-radius:10px;font-size:16px;font-weight:700;margin-top:24px;cursor:pointer;text-decoration:none;}" +
    ".btn:disabled{opacity:.6;cursor:wait;}" +
    ".btn2{display:block;width:100%;box-sizing:border-box;text-align:center;background:#fff;color:#047857;border:1.5px solid #A7F3D0;padding:13px;border-radius:10px;font-size:15px;font-weight:700;margin-top:12px;text-decoration:none;}" +
    ".note{font-size:13px;color:#6B7280;line-height:1.5;margin:16px 0 0;}" +
    ".foot{text-align:center;font-size:13px;color:#9CA3AF;margin-top:20px;}" +
    ".card{background:#fff;border-radius:16px;padding:28px 24px;box-shadow:0 4px 24px rgba(17,17,17,0.06);}";
}

function page(title, bodyHtml) {
  return "<!DOCTYPE html><html><head><meta charset='utf-8'><style>" + css_() + "</style></head>" +
    "<body><div class='wrap'><div class='card' style='text-align:center;'>" +
    "<h1>" + title + "</h1>" + bodyHtml +
    "</div></div></body></html>";
}

function formHtml(pack, info, actionUrl, lang) {
  lang = lang || "fr";
  const L = L_[lang];
  const collabField = info.collaborateurs ?
    "<label for='collaborateurs'>" + L.collab + "</label>" +
    "<input type='text' id='collaborateurs' name='collaborateurs' placeholder='" + L.collabEx + "' required>"
    : "";
  const suite = info.lien ? L.subLien : L.subDevis;

  return "<!DOCTYPE html><html lang='" + lang + "'><head><meta charset='utf-8'><style>" + css_() + "</style></head>" +
    "<body><div class='wrap'><div class='card' id='carte'>" +
    "<span class='badge'>" + L.badge(info.nom, prixAffiche_(info, lang)) + "</span>" +
    "<h1>" + L.h1(info.nom) + "</h1>" +
    "<p class='sub'>" + suite + "</p>" +
    "<form id='f' onsubmit='return envoyer(event)' data-envoi='" + L.envoi + "' data-bouton='" + L.envoyer + "' data-erreur='" + L.erreurGenerale.replace(/'/g, "&#39;") + "'>" +
    "<input type='hidden' name='pack' value='" + pack + "'>" +
    "<input type='hidden' name='lang' value='" + lang + "'>" +
    "<label for='entreprise'>" + L.entreprise + "</label>" +
    "<input type='text' id='entreprise' name='entreprise' required>" +
    "<label for='nom'>" + L.nom + "</label>" +
    "<input type='text' id='nom' name='nom' required>" +
    "<label for='telephone'>" + L.tel + "</label>" +
    "<input type='tel' id='telephone' name='telephone' placeholder='+225 ...' required>" +
    "<label for='email_actuel'>" + L.emailActuel + "</label>" +
    "<input type='email' id='email_actuel' name='email_actuel' required>" +
    "<label for='email_pro'>" + L.emailPro + "</label>" +
    "<input type='text' id='email_pro' name='email_pro' placeholder='contact@votreboite.ci' required>" +
    collabField +
    "<button class='btn' type='submit'>" + L.envoyer + "</button>" +
    "</form>" +
    "<p class='foot'>" + L.foot + "</p>" +
    "</div></div><script>" + SCRIPT_FORM_ + "</script></body></html>";
}

// =====================================================================
// NEWSLETTER : guide PDF envoyé à chaque nouvel inscrit
// =====================================================================
// À FAIRE UNE FOIS : mets le PDF du guide dans ton Google Drive, ouvre-le, et copie dans l'adresse la partie
// entre « /d/ » et « /view » (c'est l'identifiant du fichier). Colle-la ci-dessous.
const GUIDE_PDF_ID = "";      // guide en français
const GUIDE_PDF_ID_EN = "";   // guide en anglais (laisse vide tant qu'il n'existe pas : le guide français sera envoyé avec une note)
const SITE_URL = "https://www.mailpro.ci";
const NEWSLETTER_MAX_PAR_JOUR = 40;   // protège ton quota d'e-mails contre les inscriptions automatiques

// À lancer UNE FOIS depuis l'éditeur (bouton Exécuter) : autorise l'accès à Drive et vérifie que le guide est bien trouvé
function testerGuide() {
  [["français", GUIDE_PDF_ID], ["anglais", GUIDE_PDF_ID_EN]].forEach(function (g) {
    if (!g[1]) { Logger.log("⚠️ Guide " + g[0] + " : identifiant vide."); return; }
    const f = DriveApp.getFileById(g[1]);
    Logger.log("✅ Guide " + g[0] + " trouvé : " + f.getName() + " (" + Math.round(f.getSize() / 1024) + " Ko)");
  });
}

function escHtml_(s) {
  return String(s === undefined || s === null ? "" : s).replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/'/g, "&#39;").replace(/"/g, "&quot;");
}

function feuilleNewsletter_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let f = ss.getSheetByName("Newsletter");
  if (!f) {
    f = ss.insertSheet("Newsletter");
    f.appendRow(["Date", "Prénom / nom", "Email", "Source", "Statut", "Guide envoyé le", "Langue"]);
  } else if (!f.getRange(1, 7).getValue()) {
    f.getRange(1, 7).setValue("Langue");
  }
  return f;
}

// Jeton secret qui prouve que le lien de désinscription vient bien de nous
function jetonDesabo_(email) {
  const props = PropertiesService.getScriptProperties();
  let secret = props.getProperty("SECRET_DESABO");
  if (!secret) { secret = Utilities.getUuid() + Utilities.getUuid(); props.setProperty("SECRET_DESABO", secret); }
  return Utilities.computeHmacSha256Signature(email, secret)
    .map(function (b) { return ("0" + (b & 0xFF).toString(16)).slice(-2); }).join("").substring(0, 32);
}

function lienDesabo_(email, lang) {
  return ScriptApp.getService().getUrl() + "?action=desabo&e=" + encodeURIComponent(email) + "&t=" + jetonDesabo_(email) + (lang === "en" ? "&lang=en" : "");
}

function newsletterHtml_(source, lang) {
  lang = lang || "fr";
  const L = L_[lang];
  return "<!DOCTYPE html><html lang='" + lang + "'><head><meta charset='utf-8'><style>" + css_() + "</style></head>" +
    "<body><div class='wrap'><div class='card' id='carte'>" +
    "<span class='badge'>" + L.nlBadge + "</span>" +
    "<h1>" + L.nlH1 + "</h1>" +
    "<p class='sub'>" + L.nlSub + "</p>" +
    "<form id='f' onsubmit='return inscrire(event)' data-envoi='" + L.envoi + "' data-bouton='" + L.nlBouton + "' data-erreur='" + L.erreurGenerale.replace(/'/g, "&#39;") + "'>" +
    "<input type='hidden' name='source' value='" + escHtml_(source) + "'>" +
    "<input type='hidden' name='lang' value='" + lang + "'>" +
    "<div style='position:absolute;left:-9999px' aria-hidden='true'><input type='text' name='site' tabindex='-1' autocomplete='off'></div>" +
    "<label for='nom'>" + L.nlPrenom + "</label><input type='text' id='nom' name='nom' required>" +
    "<label for='email'>" + L.nlEmail + "</label><input type='email' id='email' name='email' required>" +
    "<button class='btn' type='submit'>" + L.nlBouton + "</button>" +
    "</form>" +
    "<p class='foot'>" + L.nlFoot + "</p>" +
    "</div></div><script>" + SCRIPT_NL_ + "</script></body></html>";
}

const SCRIPT_NL_ = `
function inscrire(ev) {
  ev.preventDefault();
  var f = document.getElementById('f');
  var b = f.querySelector('button');
  b.disabled = true;
  b.textContent = f.dataset.envoi;
  google.script.run
    .withSuccessHandler(function (r) {
      if (r && r.ok) {
        document.getElementById('carte').innerHTML = '<div style="text-align:center"><h1>' + r.titre + '</h1>' + r.corps + '</div>';
        window.scrollTo(0, 0);
      } else {
        b.disabled = false;
        b.textContent = f.dataset.bouton;
        alert((r && r.message) || f.dataset.erreur);
      }
    })
    .withFailureHandler(function () {
      b.disabled = false;
      b.textContent = f.dataset.bouton;
      alert(f.dataset.erreur);
    })
    .inscrireNewsletter(f);
  return false;
}
`;

// Appelée par le formulaire : enregistre l'inscrit et lui envoie le guide (dans sa langue)
function inscrireNewsletter(form) {
  form = form || {};
  const lang = langue_(form.lang);
  const L = L_[lang];
  try {
    if (form.site) return { ok: true, titre: L.merci, corps: "<p>" + L.merci + "</p>" };   // champ piège : les robots croient avoir réussi

    const email = String(form.email || "").trim().toLowerCase();
    const nom = String(form.nom || "").trim().substring(0, 80);
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email) || email.length > 120) return { ok: false, message: L.nlEmailInvalide };

    // Guide anglais si disponible, sinon le guide français (avec une note dans l'e-mail)
    const guideId = (lang === "en" && GUIDE_PDF_ID_EN) ? GUIDE_PDF_ID_EN : GUIDE_PDF_ID;
    const guideFr = lang === "en" && !GUIDE_PDF_ID_EN;
    if (!guideId) {
      try { MailApp.sendEmail(EMAIL_NOTIFICATION, "MailPro — GUIDE_PDF_ID manquant", "Quelqu'un s'est inscrit à la newsletter mais l'identifiant du guide est vide dans le code. Renseigne-le."); } catch (err) {}
      return { ok: false, message: L.nlIndispo };
    }

    const lock = LockService.getScriptLock();
    lock.waitLock(15000);
    try {
      const f = feuilleNewsletter_();
      const data = f.getDataRange().getValues();
      let ligne = -1;
      for (let i = 1; i < data.length; i++) {
        if (String(data[i][2]).toLowerCase() === email) { ligne = i + 1; break; }
      }
      if (ligne > 0 && data[ligne - 1][4] !== "Désinscrit") {
        return { ok: true, titre: L.nlDejaTitre, corps: L.nlDeja(escHtml_(email)) };
      }

      // Plafond quotidien (protège ton quota d'e-mails)
      const props = PropertiesService.getScriptProperties();
      const jour = Utilities.formatDate(new Date(), "Africa/Abidjan", "yyyy-MM-dd");
      const [jourStocke, compte] = String(props.getProperty("NL_JOUR") || "").split(":");
      const nb = jourStocke === jour ? Number(compte) || 0 : 0;
      if (nb >= NEWSLETTER_MAX_PAR_JOUR) return { ok: false, message: L.nlPlafond };

      const titre = L.nlGuideTitre;
      const pdf = DriveApp.getFileById(guideId).getBlob().setName("Guide MailPro - " + titre + ".pdf");
      MailApp.sendEmail({
        to: email,
        name: "MailPro.ci",
        subject: L.nlMailSujet(titre),
        body: L.nlMailCorps(nom, titre, SITE_URL, WHATSAPP, lienDesabo_(email, lang), guideFr),
        attachments: [pdf]
      });

      const maintenant = new Date();
      if (ligne > 0) {
        f.getRange(ligne, 5).setValue("Actif");
        f.getRange(ligne, 6).setValue(maintenant);
        f.getRange(ligne, 7).setValue(lang.toUpperCase());
      } else {
        f.appendRow([maintenant, sur_(nom), sur_(email), sur_(form.source), "Actif", maintenant, lang.toUpperCase()]);
      }
      props.setProperty("NL_JOUR", jour + ":" + (nb + 1));
    } finally {
      lock.releaseLock();
    }

    return { ok: true, titre: L.nlOkTitre, corps: L.nlOk(escHtml_(L.nlGuideTitre), escHtml_(email)) + "<a class='btn' href='" + SITE_URL + "' target='_blank' rel='noopener'>" + L.nlOffres + "</a>" };
  } catch (err) {
    return { ok: false, message: L.erreurGenerale };
  }
}

// Lien de désinscription reçu par e-mail
function desinscription_(e) {
  const lang = langue_(e.parameter.lang);
  const L = L_[lang];
  const email = String(e.parameter.e || "").trim().toLowerCase();
  let corps, titre;
  if (!email || e.parameter.t !== jetonDesabo_(email)) {
    titre = L.desaboKoTitre;
    corps = L.desaboKo;
  } else {
    const f = feuilleNewsletter_();
    const data = f.getDataRange().getValues();
    for (let i = 1; i < data.length; i++) {
      if (String(data[i][2]).toLowerCase() === email) f.getRange(i + 1, 5).setValue("Désinscrit");
    }
    titre = L.desaboOkTitre;
    corps = L.desaboOk(escHtml_(email));
  }
  return HtmlService.createHtmlOutput(page(titre, corps)).addMetaTag("viewport", "width=device-width, initial-scale=1");
}
