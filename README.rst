keycloak plugin for `Tutor <https://docs.tutor.edly.io>`__
##########################################################

Local Keycloak IdP for developing Open edX SAML SSO without a real IdP.
**Development only**: adds a ``keycloak`` service to ``tutor dev``, never to ``tutor local``.

What you get
************

- Keycloak container, auto-imported realm (default ``openedx``) with a SAML client for the LMS.
- SAML attributes released: ``firstName``, ``lastName``, ``email``, ``username``, ``attr_user_permanent_id``
  (``attr_user_permanent_id`` is the Keycloak user ID, so self-registered users work too).
- Test user (default ``testuser`` / ``tutor config printvalue KEYCLOAK_TEST_PASSWORD``).
- ``keycloak-setup`` do-command (dev only) creating the SAML SP config + ``keycloak`` IdP provider (attribute mapping included) and pulling IdP metadata.

Installation
************

.. code-block:: bash

    pip install tutor-contrib-keycloak

Usage
*****

.. code-block:: bash

    tutor plugins enable keycloak
    tutor config save
    tutor dev launch
    tutor dev do keycloak-setup   # SAML provider in the LMS; needs Keycloak up

Then:

- Keycloak admin: http://auth.local.openedx.io:8080 (``admin`` / ``tutor config printvalue KEYCLOAK_ADMIN_PASSWORD``)
- LMS login: http://local.openedx.io:8000/auth/login/tpa-saml/?auth_entry=login&idp=keycloak
- SP metadata: http://local.openedx.io:8000/auth/saml/metadata.xml

``*.local.openedx.io`` resolves to 127.0.0.1; the compose network alias makes the same
hostname work from inside the LMS container (needed to fetch IdP metadata).

Settings (``tutor config save --set KEYCLOAK_X=...``)
*****************************************************

========================================  ==========================================
``KEYCLOAK_DOCKER_IMAGE``                 ``quay.io/keycloak/keycloak:26.4``
``KEYCLOAK_HOST`` / ``KEYCLOAK_PORT``     ``auth.{{ LMS_HOST }}`` / ``8080``
``KEYCLOAK_REALM``                        ``openedx``
``KEYCLOAK_SP_ENTITY_ID``                 ``openedx``
``KEYCLOAK_REGISTRATION_ALLOWED``         ``true``
``KEYCLOAK_ADMIN_USER`` / ``_PASSWORD``   ``admin`` / generated
``KEYCLOAK_TEST_USER`` / ``_PASSWORD``    ``testuser`` / ``tutor config printvalue KEYCLOAK_TEST_PASSWORD``
``KEYCLOAK_USERNAME_ATTR``                ``username``
``KEYCLOAK_PERMANENT_ID_ATTR``            ``attr_user_permanent_id``
========================================  ==========================================

Notes
*****

- ``KEYCLOAK_REGISTRATION_ALLOWED`` shows a "Register" link on the Keycloak login page.
- Permanent ID: the LMS links an IdP identity to an account by this value. If the IdP sends no such attribute the login
  fails (``Invalid value for parameter attr_user_permanent_id``). Set ``KEYCLOAK_PERMANENT_ID_ATTR`` to an empty string
  to let the LMS fall back to the ``uid`` OID attribute, then the SAML NameID (this client uses NameID format
  ``username``, so a renamed user would look like a new person; ``persistent`` is safer). Re-run
  ``tutor dev do keycloak-setup`` after changing it.

License
*******

This software is licensed under the terms of the AGPLv3.
