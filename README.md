# 🐐 React Testing Library — Tests d'Intégration & Comportementaux

React Testing Library part d'un principe simple :
*"Plus vos tests ressemblent à la façon dont votre logiciel est utilisé,
plus ils vous donnent confiance."*

Ce repo documente mon implémentation de tests d'intégration React avec RTL —
des tests qui simulent de vraies interactions utilisateur, pas des appels
de méthodes internes de composants.

---

## Ce que testent ces tests (et ce qu'ils ne testent pas)

```
  ✅ Ce que RTL teste bien
  ───────────────────────────────────────────────────────────
  L'utilisateur voit-il le bon texte après une action ?
  Le formulaire affiche-t-il une erreur si un champ est vide ?
  Le bouton est-il désactivé pendant le chargement ?
  La page navigue-t-elle vers le bon endroit après login ?
  L'API est-elle appelée avec les bons paramètres ?

  ❌ Ce que RTL ne teste PAS (intentionnellement)
  ───────────────────────────────────────────────────────────
  Le state interne d'un composant
  L'ordre de rendu des hooks
  Les noms de variables ou de fonctions internes
  Les détails d'implémentation qui ne changent pas l'UX
```

---

## Exemple : test d'intégration d'un formulaire complet

```jsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { rest } from 'msw'
import { setupServer } from 'msw/node'
import App from './App'

const server = setupServer(
  rest.post('/api/profile', (req, res, ctx) =>
    res(ctx.json({ message: 'Profil mis à jour' }))
  )
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('Profile Update Form', () => {
  it('submits the form and shows success message', async () => {
    const user = userEvent.setup()  // userEvent v14 : setup() requis
    render(<App />)

    // Naviguer vers la page profil
    await user.click(screen.getByRole('link', { name: /mon profil/i }))

    // Remplir le formulaire
    await user.clear(screen.getByLabelText(/nom/i))
    await user.type(screen.getByLabelText(/nom/i), 'Abderrahmane Elouafi')
    await user.clear(screen.getByLabelText(/email/i))
    await user.type(screen.getByLabelText(/email/i), 'abderrahmane@test.com')

    // Soumettre
    await user.click(screen.getByRole('button', { name: /enregistrer/i }))

    // Vérifier le feedback
    expect(
      await screen.findByText(/profil mis à jour/i)
    ).toBeInTheDocument()
  })

  it('shows validation error for empty name', async () => {
    const user = userEvent.setup()
    render(<App />)
    await user.click(screen.getByRole('link', { name: /mon profil/i }))

    // Vider le champ nom et soumettre
    await user.clear(screen.getByLabelText(/nom/i))
    await user.click(screen.getByRole('button', { name: /enregistrer/i }))

    expect(screen.getByText(/le nom est requis/i)).toBeInTheDocument()
    // Vérifier que l'API N'a PAS été appelée
    expect(screen.queryByText(/profil mis à jour/i)).not.toBeInTheDocument()
  })

  it('handles API error gracefully', async () => {
    server.use(
      rest.post('/api/profile', (req, res, ctx) =>
        res(ctx.status(500), ctx.json({ message: 'Erreur serveur' }))
      )
    )
    const user = userEvent.setup()
    render(<App />)
    await user.click(screen.getByRole('link', { name: /mon profil/i }))
    await user.type(screen.getByLabelText(/nom/i), 'Test')
    await user.click(screen.getByRole('button', { name: /enregistrer/i }))

    expect(
      await screen.findByRole('alert')
    ).toHaveTextContent(/une erreur est survenue/i)
  })
})
```

---

## Sélecteurs par ordre de priorité (bonnes pratiques RTL)

```
  Priorité 1 — Accessibilité (recommandé)
  ├── getByRole('button', { name: /submit/i })
  ├── getByLabelText(/email/i)
  └── getByPlaceholderText(/rechercher/i)

  Priorité 2 — Contenu visible
  ├── getByText(/bienvenue/i)
  └── getByDisplayValue('valeur actuelle')

  Priorité 3 — Test ID (à éviter si possible)
  └── getByTestId('submit-button')  ← en dernier recours seulement
```

Utiliser `getByRole` en priorité force à écrire des composants accessibles —
les tests et l'accessibilité s'améliorent ensemble.

---

## Lancer les tests

```bash
git clone https://github.com/elouafi-abderrahmane-2002/react-testing-library.git
cd react-testing-library
npm install

npm test                        # mode watch interactif
npm test -- --coverage          # rapport de couverture HTML
npm test -- --watchAll=false    # CI / run unique
```

---

## Ce que ce projet m'a appris

`userEvent` vs `fireEvent` : les deux simulent des interactions utilisateur,
mais `userEvent` est bien plus réaliste. `fireEvent.click()` déclenche un simple
événement click. `userEvent.click()` simule tout ce qu'un vrai clic génère :
mousedown, focus, mouseup, click — dans le bon ordre. Pour des composants avec
des validations onBlur ou des états de focus, `fireEvent` peut passer des tests
qui échouent en production réelle. `userEvent` est presque toujours le bon choix.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
