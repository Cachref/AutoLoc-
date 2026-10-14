# Stratégie de fetch et de cascade — AutoLoc

Règle générale : **tout est en `FetchType.LAZY`** (déclaré explicitement) afin d'éviter de charger
toute la chaîne d'objets liés. Une cascade n'est appliquée que lorsque l'enfant ne peut pas vivre
sans son parent (composition). `@Data` n'est jamais utilisé (risque de `StackOverflowError` via
`toString`/`hashCode` sur les associations bidirectionnelles).

| Association | Côté propriétaire (FK) | Fetch | Cascade | Justification |
|---|---|---|---|---|
| Contrat → Paiement | `Paiement.id_contrat` | LAZY | ALL + orphanRemoval | Un paiement n'existe que rattaché à son contrat (composition) : supprimer le contrat supprime ses paiements, et retirer un paiement de la liste le supprime en base. LAZY : la liste n'est chargée que si on l'utilise. |
| Agence → Vehicule | `Vehicule.id_agence` | LAZY | Aucune | Un véhicule survit à la suppression de son agence (il peut être réaffecté) ; une cascade REMOVE détruirait le parc et son historique. |
| Agence → Employe | `Employe.id_agence` | LAZY | Aucune | Un employé a une existence propre ; supprimer une agence ne doit pas supprimer des personnes. Pas de chargement systématique des employés. |
| Vehicule ↔ Equipement | `vehicule_equipement` (table de jointure, côté Vehicule) | LAZY | Aucune | Les équipements sont partagés entre plusieurs véhicules : supprimer un véhicule ne doit pas supprimer un équipement utilisé ailleurs. `Set` pour éviter les doublons. |
| Client → Reservation | `Reservation.id_client` | LAZY | PERSIST | Enregistrer un client avec ses nouvelles réservations les sauvegarde en une fois. Pas de REMOVE : l'historique des réservations doit être conservé. |
| Reservation → Vehicule | `Reservation.id_vehicule` | LAZY | Aucune | Le véhicule est une entité indépendante, simplement référencée par la réservation. |
| Reservation ↔ Contrat | `Contrat.id_reservation` (unique) | LAZY | ALL (côté Reservation) | Le contrat découle de la réservation et n'a pas de sens sans elle : son cycle de vie suit celui de la réservation. Remarque : côté inverse d'un `@OneToOne`, Hibernate ne peut généralement pas créer de proxy et charge l'objet immédiatement malgré LAZY ; le côté propriétaire (`Contrat.reservation`) reste réellement LAZY. |
| Vehicule → Maintenance | `Maintenance.id_vehicule` | LAZY | PERSIST | Les maintenances sont créées avec le véhicule ; pas de REMOVE afin de garder l'historique d'entretien. |

## Remarques

- **LazyInitializationException** : accéder à une collection LAZY hors transaction la lève ; on la
  traitera par des requêtes dédiées (Atelier 7) plutôt qu'en passant en EAGER.
- **Côté inverse** : chaque `@OneToMany` utilise `mappedBy`, donc aucune colonne ni table de jointure
  supplémentaire n'est créée (seule `vehicule_equipement` existe, pour le `@ManyToMany`).
