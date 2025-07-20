openapi: 3.0.3
info:
  title: Marketplace BTP – API
  version: "1.0.0"
  description: >-
    API pour gérer :
    - Produits (bulk upload, photos, specs)
    - Panier & options de livraison
    - Commandes, bons de livraison & signature
    - Paiements variés & justificatifs
    - Historique acheteur, suivi stock, abonnement vendeur
servers:
  - url: https://api.exemple.com/v1
    description: Serveur de production
paths:

  /products/template:
    get:
      summary: Télécharger le template Excel de publication
      tags: [Produits]
      responses:
        "200":
          description: Fichier XLSX
          content:
            application/vnd.openxmlformats-officedocument.spreadsheetml.sheet:
              schema:
                type: string
                format: binary

  /products/bulk-upload:
    post:
      summary: Importer en masse des produits via fichier
      tags: [Produits]
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
      responses:
        "201":
          description: Produits créés / mis à jour

  /cart/shipping-options:
    post:
      summary: Calculer les options de livraison (séparées & consolidée)
      tags: [Panier]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CartRequest"
      responses:
        "200":
          description: Options de livraison
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ShippingOptions"

  /checkout:
    post:
      summary: Finaliser la commande
      tags: [Panier]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CheckoutRequest"
      responses:
        "201":
          description: Commande créée
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"

  /orders:
    get:
      summary: Liste des commandes de l’acheteur connecté
      tags: [Commandes]
      parameters:
        - in: query
          name: page
          schema:
            type: integer
          description: Numéro de page (pagination)
      responses:
        "200":
          description: Liste paginée
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items:
                      $ref: "#/components/schemas/OrderSummary"
                  total:
                    type: integer

  /orders/{orderId}:
    get:
      summary: Détail d’une commande (ligne, livraison, paiements)
      tags: [Commandes]
      parameters:
        - in: path
          name: orderId
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: Détails
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/OrderDetail"

  /orders/{orderId}/delivery-note:
    get:
      summary: Télécharger le bon de livraison (PDF)
      tags: [Commandes]
      parameters:
        - $ref: "#/components/parameters/orderIdParam"
      responses:
        "200":
          description: PDF du bon
          content:
            application/pdf:
              schema:
                type: string
                format: binary

  /orders/{orderId}/payments:
    get:
      summary: Historique des paiements pour une commande
      tags: [Paiements]
      parameters:
        - $ref: "#/components/parameters/orderIdParam"
      responses:
        "200":
          description: Liste de paiements
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Payment"

  /payments/initiate:
    post:
      summary: Initier un paiement
      tags: [Paiements]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/PaymentInitiate"
      responses:
        "200":
          description: Url de paiement / token
          content:
            application/json:
              schema:
                type: object
                properties:
                  paymentUrl:
                    type: string
                  paymentToken:
                    type: string

  /payments/webhook:
    post:
      summary: Recevoir les webhooks de paiements
      tags: [Paiements]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
      responses:
        "200":
          description: OK

  /payments/{paymentId}/proof:
    post:
      summary: Uploader une preuve de paiement manuel (virement / chèque)
      tags: [Paiements]
      parameters:
        - in: path
          name: paymentId
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
                uploadedBy:
                  type: integer
      responses:
        "201":
          description: Preuve enregistrée

  /admin/payments/{paymentId}/proofs:
    get:
      summary: Lister les preuves pour validation (admin only)
      tags: [Admin]
      parameters:
        - in: path
          name: paymentId
          required: true
          schema:
            type: integer
      security:
        - ApiKeyAuth: []
      responses:
        "200":
          description: Liste des proofs
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/PaymentProof"

  /admin/payments/{paymentId}/proofs/{proofId}/validate:
    post:
      summary: Valider ou refuser une preuve (admin only)
      tags: [Admin]
      parameters:
        - $ref: "#/components/parameters/paymentIdParam"
        - in: path
          name: proofId
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                status:
                  type: string
                  enum: [validated, rejected]
                rejectedReason:
                  type: string
      security:
        - ApiKeyAuth: []
      responses:
        "200":
          description: Mise à jour du statut

  /admin/payments/{paymentId}/confirm:
    post:
      summary: Confirmation manuelle de paiement pour déclencher la livraison
      tags: [Admin]
      parameters:
        - $ref: "#/components/parameters/paymentIdParam"
      security:
        - ApiKeyAuth: []
      responses:
        "200":
          description: Livraison planifiée

  /deliveries/available-carriers:
    get:
      summary: Obtenir les transporteurs disponibles les plus proches
      tags: [Livraisons]
      parameters:
        - in: query
          name: pickup_lat
          required: true
          schema: { type: number }
        - in: query
          name: pickup_lng
          required: true
          schema: { type: number }
        - in: query
          name: volume
          required: true
          schema: { type: number }
        - in: query
          name: weight
          required: true
          schema: { type: number }
        - in: query
          name: max_radius
          schema: { type: number, default: 10 }
      responses:
        "200":
          description: Liste de transporteurs
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/CarrierOption"

  /transporters/{id}/location:
    post:
      summary: Mettre à jour la position GPS du transporteur
      tags: [Transporteurs]
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: integer }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                lat: { type: number }
                lng: { type: number }
      responses:
        "200":
          description: Position enregistrée

components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

  parameters:
    orderIdParam:
      in: path
      name: orderId
      required: true
      schema:
        type: integer
    paymentIdParam:
      in: path
      name: paymentId
      required: true
      schema:
        type: integer

  schemas:
    CartItem:
      type: object
      properties:
        productId:
          type: integer
        quantity:
          type: integer

    CartRequest:
      type: object
      properties:
        items:
          type: array
          items:
            $ref: "#/components/schemas/CartItem"
        deliveryAddress:
          $ref: "#/components/schemas/Address"

    Address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        lat: { type: number }
        lng: { type: number }

    ShippingOption:
      type: object
      properties:
        separate:
          type: array
          items:
            $ref: "#/components/schemas/CarrierOption"
        combined:
          $ref: "#/components/schemas/CombinedOption"

    CarrierOption:
      type: object
      properties:
        carrierId: { type: integer }
        name: { type: string }
        mode: { type: string }
        distance_km: { type: number }
        eta_min: { type: number }
        cost_est: { type: number }

    CombinedOption:
      type: object
      properties:
        route:
          type: array
          items: { type: integer }
        mode: { type: string }
        cost: { type: number }
        etaMin: { type: number }

    CheckoutRequest:
      type: object
      properties:
        cartId: { type: integer }
        shippingOption:
          oneOf:
            - $ref: "#/components/schemas/CarrierOption"
            - $ref: "#/components/schemas/CombinedOption"
        paymentMethod: { type: string }

    OrderSummary:
      type: object
      properties:
        id: { type: integer }
        created_at: { type: string, format: date-time }
        total_ttc: { type: number }
        status: { type: string }

    OrderItem:
      type: object
      properties:
        productId: { type: integer }
        quantity: { type: integer }
        unit_price_ht: { type: number }

    Order:
      type: object
      properties:
        id: { type: integer }
        buyer_id: { type: integer }
        created_at: { type: string, format: date-time }
        total_ttc: { type: number }
        status: { type: string }
        items:
          type: array
          items:
            $ref: "#/components/schemas/OrderItem"

    OrderDetail:
      allOf:
        - $ref: "#/components/schemas/Order"
        - type: object
          properties:
            delivery_note:
              $ref: "#/components/schemas/DeliveryNote"
            payments:
              type: array
              items:
                $ref: "#/components/schemas/Payment"

    DeliveryNote:
      type: object
      properties:
        id: { type: integer }
        document_url: { type: string, format: uri }
        issued_at: { type: string, format: date-time }
        status: { type: string }
        signed_at: { type: string, format: date-time }
        signer_name: { type: string }
        signature_url: { type: string, format: uri }

    PaymentInitiate:
      type: object
      properties:
        orderId: { type: integer }
        method: { type: string }
        amount: { type: number }
        returnUrl: { type: string }

    Payment:
      type: object
      properties:
        id: { type: integer }
        order_id: { type: integer }
        method: { type: string }
        provider: { type: string }
        amount: { type: number }
        currency: { type: string }
        status: { type: string }
        transaction_id: { type: string }
        created_at: { type: string, format: date-time }
        updated_at: { type: string, format: date-time }

    PaymentProof:
      type: object
      properties:
        id: { type: integer }
        payment_id: { type: integer }
        uploaded_by: { type: integer }
        proof_url: { type: string, format: uri }
        file_name: { type: string }
        uploaded_at: { type: string, format: date-time }
        status: { type: string }
        rejected_reason: { type: string }

security:
  - ApiKeyAuth: []
