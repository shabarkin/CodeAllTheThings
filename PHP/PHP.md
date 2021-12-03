# PHP

# Summary

## Access Controls & Authorization

- **#CodeIgniter**
    
    > CodeIgniter is a powerful PHP framework with a very small footprint, built for developers who need a simple and elegant toolkit to create full-featured web applications. CodeIgniter encourages MVC, but does not force it on you. [https://codeigniter.com/](https://codeigniter.com/)
    > 
    
    Walk through all the files within the `controllers` folder and identify all the public functions, then review the endpoint authorization mechanisms, by default all the public functions on the controller are accessible to the web application. 
    
    ```php
    public function
    ```
    
    ```bash
    grep -HanriE "public function [a-z0-9_]+\\("
    ```
    
    ```bash
    gf -save php-authorization -HanriE "public function [a-z0-9_]+\\("
    ```