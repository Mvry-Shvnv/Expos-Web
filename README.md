#Explorateur de fichier
Un explorateur de fichiers (comme l'Explorateur de fichiers sur Windows, Finder sur macOS, ou Nautilus/Files sur Linux) est un outil essentiel de votre système d'exploitation pour gérer, organiser et interagir avec vos fichiers et dossiers.



from tkinter import *
from tkinter import ttk
from tkinter import filedialog
from tkinter import simpledialog, messagebox
import os
import webbrowser
from datetime import datetime
import shutil


class FileManager:
    def __init__(self, root):
        self.root = root
        self.favorites = []
        self.recent_folders = []
        self.current_folder = ""
        self.selected_item = ""
        self.clipboard = None
        self.setup_ui()
        self.load_favorites()

    def setup_ui(self):
        self.root.title("Gestionnaire de fichiers")
        self.root.geometry("1200x700")
        self.root.minsize(800, 600)
        self.root.configure(highlightthickness=4, highlightbackground="black")

        # Configuration de la grille principale
        self.root.columnconfigure(0, weight=1)  # Colonne gauche (navigation)
        self.root.columnconfigure(1, weight=5)  # Colonne droite (contenu)
        self.root.rowconfigure(0, weight=0)  # Barre de chemin
        self.root.rowconfigure(1, weight=0)  # Barre de recherche
        self.root.rowconfigure(2, weight=1)  # Contenu principal

        # Barre de chemin (ligne 0)
        self.setup_path_bar()

        # Barre de recherche (ligne 1)
        self.setup_search_bar()

        # Frame de navigation gauche (ligne 2)
        self.setup_navigation_frame()

        # Frame de contenu droit (ligne 2)
        self.setup_content_frame()

    def setup_path_bar(self):
        path_frame = ttk.Frame(self.root, padding=5)
        path_frame.grid(column=0, row=0, columnspan=2, sticky=(N, S, E, W))

        self.path_entry = ttk.Entry(path_frame)
        self.path_entry.pack(side=LEFT, fill=X, expand=True, padx=(0, 10))

        browse_button = ttk.Button(path_frame, text="Parcourir", command=self.browse_folder)
        browse_button.pack(side=LEFT)

        new_menu = Menu(self.root, tearoff=0)
        new_menu.add_command(label="Dossier", command=self.create_new_folder)
        new_menu.add_command(label="Fichier texte", command=lambda: self.create_new_file(".txt"))
        new_menu.add_command(label="Fichier Word", command=lambda: self.create_new_file(".docx"))
        new_menu.add_command(label="Fichier Python", command=lambda: self.create_new_file(".py"))

        new_button = ttk.Menubutton(path_frame, text="Nouveau", direction="below")
        new_button["menu"] = new_menu
        new_button.pack(side=LEFT, padx=(10, 0))

    def setup_search_bar(self):
        search_frame = ttk.Frame(self.root, padding=5)
        search_frame.grid(column=0, row=1, columnspan=2, sticky=(N, S, E, W))

        self.search_var = StringVar()
        self.search_var.trace("w", self.on_search)
        self.search_entry = ttk.Entry(search_frame, textvariable=self.search_var)
        self.search_entry.pack(side=LEFT, fill=X, expand=True)
        self.search_entry.insert(0, "Rechercher...")

        if self.search_entry.get() == "Rechercher...":
            self.search_entry.bind("<FocusIn>", lambda e: self.search_entry.delete(0, END))

        clear_button = ttk.Button(search_frame, text="×", width=3, command=self.clear_search)
        clear_button.pack(side=LEFT, padx=(10, 0))

    def setup_navigation_frame(self):
        # Frame principal pour la navigation avec largeur fixe
        nav_frame = ttk.Frame(self.root, width=180)  # Largeur réduite
        nav_frame.grid(column=0, row=2, sticky=(N, S, E, W), padx=2, pady=5)
        nav_frame.grid_propagate(False)  # Empêche le redimensionnement automatique

        # Ajout d'une bordure
        border_frame = Frame(nav_frame, bg="black", padx=1, pady=1)
        border_frame.pack(expand=True, fill=BOTH)

        # Frame pour le contenu de navigation (sans scrollbar)
        self.nav_content = ttk.Frame(border_frame)
        self.nav_content.pack(expand=True, fill=BOTH)

        # Contenu de la navigation
        ttk.Label(self.nav_content, text="Navigation", font=('Arial', 10, 'bold')).pack(pady=(10, 5), anchor="w")
        ttk.Button(self.nav_content, text="Favoris", command=self.show_favorites).pack(fill="x", pady=2)
        ttk.Button(self.nav_content, text="Récents", command=self.show_recents).pack(fill="x", pady=2)
        ttk.Button(self.nav_content, text="Ordinateur", command=self.show_computer).pack(fill="x", pady=2)
        ttk.Button(self.nav_content, text="Tags", command=self.show_tags).pack(fill="x", pady=2)

    def setup_content_frame(self):
        # Frame principal pour le contenu
        content_frame = ttk.Frame(self.root)
        content_frame.grid(column=1, row=2, sticky=(N, S, E, W), padx=5, pady=5)

        # Ajout d'une bordure
        border_frame = Frame(content_frame, bg="black", padx=1, pady=1)
        border_frame.pack(expand=True, fill=BOTH)

        # Canvas et scrollbar pour le contenu
        self.content_canvas = Canvas(border_frame)
        self.content_scroll = ttk.Scrollbar(border_frame, orient="vertical", command=self.content_canvas.yview)
        self.scrollable_frame = ttk.Frame(self.content_canvas)

        self.scrollable_frame.bind(
            "<Configure>",
            lambda e: self.content_canvas.configure(scrollregion=self.content_canvas.bbox("all")))

        self.content_canvas.create_window((0, 0), window=self.scrollable_frame, anchor="nw")
        self.content_canvas.configure(yscrollcommand=self.content_scroll.set)

        self.content_scroll.pack(side="right", fill="y")
        self.content_canvas.pack(side="left", fill="both", expand=True)

        # Activer le défilement avec la molette
        self.content_canvas.bind_all("<MouseWheel>",
                                   lambda e: self.content_canvas.yview_scroll(int(-1 * (e.delta / 120)), "units"))

        # Menu contextuel
        self.context_menu = Menu(self.root, tearoff=0)
        self.context_menu.add_command(label="Ouvrir", command=self.open_selected)
        self.context_menu.add_command(label="Renommer", command=self.rename_selected)
        self.context_menu.add_command(label="Supprimer", command=self.delete_selected)
        self.context_menu.add_command(label="Ajouter aux favoris", command=self.add_to_favorites)
        self.context_menu.add_command(label="Copier", command=self.copy_selected)
        self.context_menu.add_command(label="Coller", command=self.paste_selected)

    def show_favorites(self):
        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        headers = ["Nom", "Chemin", "Type"]
        for col, header in enumerate(headers):
            ttk.Label(self.scrollable_frame, text=header, font=('Arial', 10, 'bold')).grid(
                row=0, column=col, sticky="ew", padx=5, pady=2)

        if not self.favorites:
            ttk.Label(self.scrollable_frame, text="Aucun favori").grid(row=1, column=0, columnspan=3)
            return

        for row, fav_path in enumerate(self.favorites, start=1):
            icon = self.get_icon(fav_path)
            fav_name = os.path.basename(fav_path)
            fav_type = "Dossier" if os.path.isdir(fav_path) else "Fichier"

            lbl = ttk.Label(self.scrollable_frame, image=icon, text=f"  {fav_name}", compound=LEFT)
            lbl.image = icon
            lbl.grid(row=row, column=0, sticky="w", padx=5, pady=2)
            lbl.bind("<Double-Button-1>", lambda e, p=fav_path: self.on_double_click(e, p))
            lbl.bind("<Button-3>", lambda e, p=fav_path: self.show_context_menu(e, p))

            ttk.Label(self.scrollable_frame, text=fav_path).grid(row=row, column=1, sticky="w", padx=5)
            ttk.Label(self.scrollable_frame, text=fav_type).grid(row=row, column=2, padx=5)

        self.scrollable_frame.columnconfigure(0, weight=2)
        self.scrollable_frame.columnconfigure(1, weight=3)
        self.scrollable_frame.columnconfigure(2, weight=1)

        self.content_canvas.configure(scrollregion=self.content_canvas.bbox("all"))

    def clear_search(self):
        self.search_var.set("")
        if hasattr(self, 'current_folder') and self.current_folder:
            self.display_folder_content(self.current_folder)

    def browse_folder(self):
        folder_path = filedialog.askdirectory()
        if folder_path:
            self.display_folder_content(folder_path)
            self.add_to_recent_folders(folder_path)

    def display_folder_content(self, folder_path):
        self.current_folder = folder_path
        self.path_entry.delete(0, END)
        self.path_entry.insert(0, folder_path)

        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        headers = ["Nom", "Taille", "Date de création"]
        for col, header in enumerate(headers):
            ttk.Label(self.scrollable_frame, text=header, font=('Arial', 10, 'bold')).grid(
                row=0, column=col, sticky="ew", padx=5, pady=2)

        try:
            items = os.listdir(folder_path)
        except PermissionError:
            ttk.Label(self.scrollable_frame, text="Accès refusé").grid(row=1, column=0, columnspan=3)
            return
        except Exception as e:
            ttk.Label(self.scrollable_frame, text=f"Erreur: {str(e)}").grid(row=1, column=0, columnspan=3)
            return

        if not items:
            ttk.Label(self.scrollable_frame, text="Dossier vide").grid(row=1, column=0, columnspan=3)
            return

        for row, item in enumerate(items, start=1):
            try:
                item_path = os.path.join(folder_path, item)
                icon = self.get_icon(item_path)

                lbl = ttk.Label(self.scrollable_frame, image=icon, text=f"  {item}", compound=LEFT)
                lbl.image = icon
                lbl.grid(row=row, column=0, sticky="w", padx=5, pady=2)
                lbl.bind("<Double-Button-1>", lambda e, p=item_path: self.on_double_click(e, p))
                lbl.bind("<Button-3>", lambda e, p=item_path: self.show_context_menu(e, p))

                ttk.Label(self.scrollable_frame, text=self.get_file_size(item_path)).grid(
                    row=row, column=1, sticky="e", padx=5)

                ttk.Label(self.scrollable_frame, text=self.get_creation_date(item_path)).grid(
                    row=row, column=2, padx=5)

            except Exception as e:
                print(f"Erreur avec {item}: {e}")
                continue

        self.scrollable_frame.columnconfigure(0, weight=3)
        self.scrollable_frame.columnconfigure(1, weight=1)
        self.scrollable_frame.columnconfigure(2, weight=1)

        self.content_canvas.configure(scrollregion=self.content_canvas.bbox("all"))

    def on_double_click(self, event, item_path):
        if os.path.isdir(item_path):
            self.display_folder_content(item_path)
            self.add_to_recent_folders(item_path)
        else:
            self.open_file(item_path)

    def show_context_menu(self, event, item_path):
        self.selected_item = item_path
        try:
            self.context_menu.tk_popup(event.x_root, event.y_root)
        finally:
            self.context_menu.grab_release()

    def open_selected(self):
        if hasattr(self, 'selected_item') and self.selected_item:
            self.on_double_click(None, self.selected_item)

    def rename_selected(self):
        if hasattr(self, 'selected_item') and self.selected_item:
            new_name = simpledialog.askstring("Renommer", "Nouveau nom:",
                                              initialvalue=os.path.basename(self.selected_item))
            if new_name:
                try:
                    new_path = os.path.join(os.path.dirname(self.selected_item), new_name)
                    os.rename(self.selected_item, new_path)
                    self.display_folder_content(os.path.dirname(self.selected_item))
                except Exception as e:
                    messagebox.showerror("Erreur", f"Impossible de renommer: {e}")

    def delete_selected(self):
        if hasattr(self, 'selected_item') and self.selected_item:
            if messagebox.askyesno("Confirmer", "Voulez-vous vraiment supprimer ce fichier/dossier?"):
                try:
                    if os.path.isdir(self.selected_item):
                        shutil.rmtree(self.selected_item)
                    else:
                        os.remove(self.selected_item)
                    self.display_folder_content(os.path.dirname(self.selected_item))
                except Exception as e:
                    messagebox.showerror("Erreur", f"Impossible de supprimer: {e}")

    def add_to_favorites(self):
        if hasattr(self, 'selected_item') and self.selected_item and os.path.isdir(self.selected_item):
            if self.selected_item not in self.favorites:
                self.favorites.append(self.selected_item)
                self.save_favorites()
                messagebox.showinfo("Info", "Ajouté aux favoris")

    def copy_selected(self):
        if hasattr(self, 'selected_item') and self.selected_item:
            self.clipboard = ('copy', self.selected_item)

    def paste_selected(self):
        if hasattr(self, 'clipboard') and self.clipboard and hasattr(self, 'current_folder') and self.current_folder:
            operation, path = self.clipboard
            new_path = os.path.join(self.current_folder, os.path.basename(path))

            try:
                if operation == 'copy':
                    if os.path.isdir(path):
                        shutil.copytree(path, new_path)
                    else:
                        shutil.copy2(path, new_path)
                self.display_folder_content(self.current_folder)
            except Exception as e:
                messagebox.showerror("Erreur", f"Impossible de coller: {e}")

    def create_new_folder(self):
        if hasattr(self, 'current_folder') and self.current_folder:
            folder_name = simpledialog.askstring("Nouveau dossier", "Nom du dossier:")
            if folder_name:
                try:
                    os.mkdir(os.path.join(self.current_folder, folder_name))
                    self.display_folder_content(self.current_folder)
                except Exception as e:
                    messagebox.showerror("Erreur", f"Impossible de créer le dossier: {e}")

    def create_new_file(self, extension):
        if hasattr(self, 'current_folder') and self.current_folder:
            file_name = simpledialog.askstring("Nouveau fichier", f"Nom du fichier (sans {extension}):")
            if file_name:
                try:
                    with open(os.path.join(self.current_folder, file_name + extension), 'w') as f:
                        pass
                    self.display_folder_content(self.current_folder)
                except Exception as e:
                    messagebox.showerror("Erreur", f"Impossible de créer le fichier: {e}")

    def on_search(self, *args):
        if hasattr(self, 'current_folder') and self.current_folder:
            search_term = self.search_var.get().lower()
            if search_term and search_term != "rechercher...":
                try:
                    items = os.listdir(self.current_folder)
                    filtered_items = [item for item in items if search_term in item.lower()]
                    self.display_filtered_content(self.current_folder, filtered_items)
                except Exception as e:
                    messagebox.showerror("Erreur", f"Erreur de recherche: {e}")

    def display_filtered_content(self, folder_path, items):
        self.current_folder = folder_path

        for widget in self.scrollable_frame.winfo_children():
            widget.destroy()

        headers = ["Nom", "Taille", "Date de création"]
        for col, header in enumerate(headers):
            ttk.Label(self.scrollable_frame, text=header, font=('Arial', 10, 'bold')).grid(
                row=0, column=col, sticky="ew", padx=5, pady=2)

        if not items:
            ttk.Label(self.scrollable_frame, text="Aucun résultat").grid(row=1, column=0, columnspan=3)
            return

        for row, item in enumerate(items, start=1):
            item_path = os.path.join(folder_path, item)
            icon = self.get_icon(item_path)

            lbl = ttk.Label(self.scrollable_frame, image=icon, text=f"  {item}", compound=LEFT)
            lbl.image = icon
            lbl.grid(row=row, column=0, sticky="w", padx=5, pady=2)
            lbl.bind("<Double-Button-1>", lambda e, p=item_path: self.on_double_click(e, p))
            lbl.bind("<Button-3>", lambda e, p=item_path: self.show_context_menu(e, p))

            ttk.Label(self.scrollable_frame, text=self.get_file_size(item_path)).grid(
                row=row, column=1, sticky="e", padx=5)

            ttk.Label(self.scrollable_frame, text=self.get_creation_date(item_path)).grid(
                row=row, column=2, padx=5)

        self.scrollable_frame.columnconfigure(0, weight=3)
        self.scrollable_frame.columnconfigure(1, weight=1)
        self.scrollable_frame.columnconfigure(2, weight=1)

        self.content_canvas.configure(scrollregion=self.content_canvas.bbox("all"))

    def show_recents(self):
        if self.recent_folders:
            self.display_folder_content(self.recent_folders[0])
        else:
            messagebox.showinfo("Info", "Aucun dossier récent")

    def show_computer(self):
        self.display_folder_content("/")

    def show_tags(self):
        messagebox.showinfo("Info", "Fonctionnalité Tags à implémenter")

    def add_to_recent_folders(self, folder_path):
        if folder_path in self.recent_folders:
            self.recent_folders.remove(folder_path)
        self.recent_folders.insert(0, folder_path)
        if len(self.recent_folders) > 10:
            self.recent_folders = self.recent_folders[:10]

    def save_favorites(self):
        try:
            with open("favorites.txt", "w") as f:
                for fav in self.favorites:
                    f.write(fav + "\n")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de sauvegarder les favoris: {e}")

    def load_favorites(self):
        try:
            if os.path.exists("favorites.txt"):
                with open("favorites.txt", "r") as f:
                    self.favorites = [line.strip() for line in f.readlines() if line.strip()]
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de charger les favoris: {e}")

    @staticmethod
    def get_icon(file_path, size=14):
        try:
            if os.path.isdir(file_path):
                return PhotoImage(file="folder.png").subsample(size, size)
            elif file_path.lower().endswith(".pdf"):
                return PhotoImage(file="pdf.png").subsample(size, size)
            elif file_path.lower().endswith((".docx", ".doc")):
                return PhotoImage(file="word.png").subsample(size, size)
            elif file_path.lower().endswith((".jpg", ".png", ".jpeg", ".gif")):
                return PhotoImage(file="image.png").subsample(size, size)
            elif file_path.lower().endswith((".rar", ".zip", ".7z")):
                return PhotoImage(file="rar-format.png").subsample(size, size)
            elif file_path.lower().endswith((".py")):
                return PhotoImage(file="python.png").subsample(size, size)
            elif file_path.lower().endswith((".txt", ".csv")):
                return PhotoImage(file="txt.png").subsample(size, size)
            elif file_path.lower().endswith((".pptx", ".ppt")):
                return PhotoImage(file="powerpoint.png").subsample(size, size)
            elif file_path.lower().endswith((".xlsx", ".xls")):
                return PhotoImage(file="excel.png").subsample(size, size)
            else:
                return PhotoImage(file="file_icon.png").subsample(size, size)
        except:
            return PhotoImage(width=1, height=1)

    @staticmethod
    def get_file_size(file_path):
        if os.path.isdir(file_path):
            return "-"
        try:
            size = os.path.getsize(file_path)
            if size < 1024:
                return f"{size} B"
            elif size < 1024 * 1024:
                return f"{size / 1024:.1f} KB"
            elif size < 1024 * 1024 * 1024:
                return f"{size / (1024 * 1024):.1f} MB"
            else:
                return f"{size / (1024 * 1024 * 1024):.1f} GB"
        except:
            return "N/A"

    @staticmethod
    def get_creation_date(file_path):
        try:
            timestamp = os.path.getctime(file_path)
            return datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M')
        except:
            return "N/A"

    @staticmethod
    def open_file(file_path):
        try:
            webbrowser.open(file_path)
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible d'ouvrir le fichier: {e}")


if __name__ == "__main__":
    root = Tk()
    app = FileManager(root)
    root.mainloop()
