================================================
 Request.el -- Easy HTTP request for Emacs Lisp
================================================

.. sidebar:: Links

   * `Documentation <http://tkf.github.com/emacs-request/>`_ (at GitHub Pages)

     * `Manual <http://tkf.github.com/emacs-request/manual.html>`_

   * `Repository <https://github.com/tkf/emacs-request>`_ (at GitHub)
   * `Issue tracker <https://github.com/tkf/emacs-request/issues>`_ (at GitHub)
   * `Travis CI <https://travis-ci.org/#!/tkf/emacs-request>`_ |build-status|
   * `MELPA <https://melpa.org/#/request>`_ |melpa-badge|
   * `MELPA Stable <https://stable.melpa.org/#/request>`_ |melpa-stable-badge|


What is it?
===========

``request.el`` is an elisp interface to HTTP requests.  It uses ``curl`` as its backend or emacs's native ``url.el`` library if ``curl`` is not found.

The default encoding for requests is ``utf-8``.  Please explicitly specify ``:encoding 'binary`` for binary data.


Installation
============

``request.el`` is available on `MELPA <https://melpa.org/>`_ and `MELPA stable <https://stable.melpa.org>`_.


Examples
========

GET:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/get"
   :params '(("key" . "value") ("key2" . "value2"))
   :parser 'json-read
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'args data)))))

POST:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/post"
   :type "POST"
   :data '(("key" . "value") ("key2" . "value2"))
   ;; :data "key=value&key2=value2"  ; this is equivalent
   :parser 'json-read
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'form data)))))

Block until completion:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/get"
   :sync t
   :complete (cl-function
             (lambda (&key response &allow-other-keys)
               (message "Done: %s" (request-response-status-code response)))))

Request binary data:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/get"
   :encoding 'binary
   :complete (cl-function
             (lambda (&key response &allow-other-keys)
               (message "Done: %s" (request-response-status-code response)))))

POST file (**WARNING**: it will send the contents of the current buffer!):

.. code:: emacs-lisp

  (request
   "http://httpbin.org/post"
   :type "POST"
   :files `(("current buffer" . ,(current-buffer))
            ("data" . ("data.csv" :data "1,2,3\n4,5,6\n")))
   :parser 'json-read
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'files data)))))

Rich callback dispatch (like `jQuery.ajax`):

.. code:: emacs-lisp

  (request
   "http://httpbin.org/status/418"     ; try other codes, for example:
   ;; "http://httpbin.org/status/200"  ; success callback will be called.
   ;; "http://httpbin.org/status/400"  ; you will see "Got 400."
   :parser 'buffer-string
   :success
   (cl-function (lambda (&key data &allow-other-keys)
                  (when data
                    (with-current-buffer (get-buffer-create "*request demo*")
                      (erase-buffer)
                      (insert data)
                      (pop-to-buffer (current-buffer))))))
   :error
   (cl-function (lambda (&rest args &key error-thrown &allow-other-keys)
                  (message "Got error: %S" error-thrown)))
   :complete (lambda (&rest _) (message "Finished!"))
   :status-code '((400 . (lambda (&rest _) (message "Got 400.")))
                  (418 . (lambda (&rest _) (message "Got 418.")))))

Flexible PARSER option:

.. code:: emacs-lisp

  (request
   "https://github.com/tkf/emacs-request/commits/master.atom"
   ;; Parse XML in response body:
   :parser (lambda () (libxml-parse-xml-region (point) (point-max)))
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               ;; Just don't look at this function....
               (let ((get (lambda (node &rest names)
                            (if names
                                (apply get
                                       (first (xml-get-children
                                               node (car names)))
                                       (cdr names))
                              (first (xml-node-children node))))))
                 (message "Latest commit: %s (by %s)"
                          (funcall get data 'entry 'title)
                          (funcall get data 'entry 'author 'name))))))

PUT JSON data:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/put"
   :type "PUT"
   :data (json-encode '(("key" . "value") ("key2" . "value2")))
   :headers '(("Content-Type" . "application/json"))
   :parser 'json-read
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'json data)))))

PUT JSON data including non-ascii strings:

.. code:: emacs-lisp

  (request
   "http://httpbin.org/put"
   :type "PUT"
   :data (json-encode '(("key" . "値1") ("key2" . "値2")))
   :headers '(("Content-Type" . "application/json"))
   :parser 'json-read
   :encoding 'utf-8
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'json data)))))

Another PUT JSON example (nested JSON using alist structure, how to represent a boolean & how to selectively evaluate lisp):

.. code:: emacs-lisp

  ;; (1) Prepend alist structure with a backtick (`) rather than single quote (')
  ;;     to allow elisp evaluation of selected elements prefixed with a comma (,)
  ;; (2) This value is expected as a boolean so use the nil / t elisp alist denotation
  ;; (3) The function will be evaluated as it has been prefixed with a comma (,)
  (request
   "http://httpbin.org/put"
   :type "PUT"
   :data (json-encode `(("jsonArray" . (("item1" . "value 1") ;; (1)
                                        ("item2" . t)         ;; (2)
                                        ("item3" . ,(your-custom-elisp-function)))))) ;; (3)
   :headers '(("Content-Type" . "application/json"))
   :parser 'json-read
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "I sent: %S" (assoc-default 'json data)))))

GET with Unix domain socket data:

.. code:: emacs-lisp

  (request
   "http:/hello.txt"
   :unix-socket "/tmp/app.sock"
   :parser (lambda () (buffer-string))
   :success (cl-function
             (lambda (&key data &allow-other-keys)
               (message "Got: %s" data))))


License
=======

Request.el is free software under GPL v3.
See COPYING file for details.


.. |build-status|
   image:: https://secure.travis-ci.org/tkf/emacs-request.svg
           ?branch=master
   :target: http://travis-ci.org/tkf/emacs-request
   :alt: Build Status
.. |melpa-badge|
   image:: http://melpa.org/packages/request-badge.svg
   :target: http://melpa.org/#/request
   :alt: MELPA Badge
.. |melpa-stable-badge|
   image:: http://stable.melpa.org/packages/request-badge.svg
   :target: http://stable.melpa.org/#/request
   :alt: MELPA Stable Badge
