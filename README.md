# Android-MVVP-SQLite
An Exemple using of Android with SQLite and MVVP architecture.

[Tutoriale](https://codinginflow.com/tutorials/android/room-viewmodel-livedata-recyclerview-mvvm).

# Android SQLite
### Setup
Under dependencies add:
```
    def lifecycle_version = "2.0.0"
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
    annotationProcessor "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
    def room_version = "2.1.0-alpha07"
    implementation "androidx.room:room-runtime:$room_version"
    annotationProcessor "androidx.room:room-compiler:$room_version"
```

_Note_: resolve compatibility [issues](https://developer.android.com/jetpack/androidx/migrate).

## Room

### Entity

GIST:
```java
@Entity(tableName = "note_table")
public class Note {

    @PrimaryKey(autoGenerate = true)
    private int id;

    //TODO Set atrb
    private String title;

    //TODO Set Ctor
    public Note(String title) {
        this.title = title;
    }
    
    public void setId(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    //TODO Set Setters && Getters
    public String getTitle() {
        return title;
    }
}
```

### Data Acc Obj
The DAO is an interface that defines all the database operations we want to do on our entity.

GIST:

```java
@Dao
public interface NoteDao {

    @Insert
    void insert(Note note);

    @Update
    void update(Note note);

    @Delete
    void delete(Note note);

    @Query("DELETE FROM note_table")
    void deleteAllNotes();

    @Query("SELECT * FROM note_table ORDER BY priority DESC")
    LiveData<List<Note>> getAllNotes();
}
```

### RoomDatabase

The `RoomDatabase` is an abstract class that ties all the pieces together and connects the entities to their corresponding DAO. 

Just as in an `SQLiteOpenHelper`, we have to define a version number and a migration strategy. 

With `fallbackToDestructiveMigration` we can let Room recreate our database if we increase the version number.

GIST:
```java
@Database(entities = {Note.class}, version = 1)
public abstract class NoteDatabase extends RoomDatabase {

    //Single instance
    private static NoteDatabase instance;

    //for Acc DAO
    public abstract NoteDao noteDao();

    public static synchronized NoteDatabase getInstance(Context context) {
        if (instance == null) {
            instance = Room.databaseBuilder(context.getApplicationContext(),
                    NoteDatabase.class, "note_database")
                    .fallbackToDestructiveMigration()
                    .build();
        }
        return instance;
    }
}
```

## Repository

The Repository ist a simple Java class that abstracts the data layer from the rest of the app and mediates between different data sources,
 like a web service and a local cache. 
 
It hides the different database operations (like SQLite queries) and provides a clean API to the ViewModel. 

Since Room doesn't allow database queries on the main thread, we use AsyncTasks to execute them asynchronously. 

LiveData is fetched on a worker thread automatically, so we don't have to take care of this.

Callback to our database builder where we populate our database in the onCreate method so we don't start with an empty table. 
We can also override onOpen if we want to execute code every time our Room database is opened.

```java
public class NoteRepository {
    private NoteDao noteDao;
    private LiveData<List<Note>> allNotes;

    public NoteRepository(Application application) {
        NoteDatabase database = NoteDatabase.getInstance(application);
        noteDao = database.noteDao();
        allNotes = noteDao.getAllNotes();
    }

    public void insert(Note note) {
        new InsertNoteAsyncTask(noteDao).execute(note);
    }

    public void update(Note note) {
        new UpdateNoteAsyncTask(noteDao).execute(note);
    }

    public void delete(Note note) {
        new DeleteNoteAsyncTask(noteDao).execute(note);
    }

    public void deleteAllNotes() {
        new DeleteAllNotesAsyncTask(noteDao).execute();
    }

    public LiveData<List<Note>> getAllNotes() {
        return allNotes;
    }

    private static class InsertNoteAsyncTask extends AsyncTask<Note, Void, Void> {
        private NoteDao noteDao;

        private InsertNoteAsyncTask(NoteDao noteDao) {
            this.noteDao = noteDao;
        }

        @Override
        protected Void doInBackground(Note... notes) {
            noteDao.insert(notes[0]);
            return null;
        }
    }

    private static class UpdateNoteAsyncTask extends AsyncTask<Note, Void, Void> {
        private NoteDao noteDao;

        private UpdateNoteAsyncTask(NoteDao noteDao) {
            this.noteDao = noteDao;
        }

        @Override
        protected Void doInBackground(Note... notes) {
            noteDao.update(notes[0]);
            return null;
        }
    }

    private static class DeleteNoteAsyncTask extends AsyncTask<Note, Void, Void> {
        private NoteDao noteDao;

        private DeleteNoteAsyncTask(NoteDao noteDao) {
            this.noteDao = noteDao;
        }

        @Override
        protected Void doInBackground(Note... notes) {
            noteDao.delete(notes[0]);
            return null;
        }
    }

    private static class DeleteAllNotesAsyncTask extends AsyncTask<Void, Void, Void> {
        private NoteDao noteDao;

        private DeleteAllNotesAsyncTask(NoteDao noteDao) {
            this.noteDao = noteDao;
        }

        @Override
        protected Void doInBackground(Void... voids) {
            noteDao.deleteAllNotes();
            return null;
        }
    }
}
```

## Viewmodel 

The ViewModel works as a gateway between the UI controller and the repository.

It stores and processes data for the activity/fragment and it doesn't get destoyed on configuration changes, 
so it doesn't lose it's variable state for example when the device is rotated.

By extending AndroidViewModel, we get a handle to the application context, which we then use to instantiate our RoomDatabase.

GIST:
```java
public class NoteViewModel extends AndroidViewModel {
    //TODO Change
    private NoteRepository repository;
    private LiveData<List<Note>> allNotes;

    //TODO Change
    public NoteViewModel(@NonNull Application application) {
        super(application);
        repository = new NoteRepository(application);
        allNotes = repository.getAllNotes();
    }

    //TODO Change
    public void insert(Note note) {
        repository.insert(note);
    }

    //TODO Change
    public void update(Note note) {
        repository.update(note);
    }
    
    //TODO Change
    public void delete(Note note) {
        repository.delete(note);
    }
    
    //TODO Change
    public void deleteAllNotes() {
        repository.deleteAllNotes();
    }

    public LiveData<List<Note>> getAllNotes() {
        //TODO Change
        return allNotes;
    }
}
```

## Adapter

GIST:
```java
public class NoteAdapter extends RecyclerView.Adapter<NoteAdapter.NoteHolder> {
    //TODO change (REFRACTOR) ARRAY with array name that hold data
    //TODO Change List TYpe
    private List<Note> ARRAY = new ArrayList<>();
    private OnItemClickListener listener;

    @NonNull
    @Override
    public NoteHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        //TODO change ITEM_SCHEMA with custom item in layout folder
        View itemView = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.ITEM_SCHEMA, parent, false);
        return new NoteHolder(itemView);
    }

    @Override
    public void onBindViewHolder(@NonNull NoteHolder holder, int position) {
        Note currentNote = ARRAY.get(position);
        //Link UI With a var 2/2 {getTitle() defined at Note.java}
        holder.textViewTitle.setText(currentNote.getTitle());
    }

    @Override
    public int getItemCount() {
        return ARRAY.size();
    }

    public void setNotes(List<Note> ARRAY) {
        this.ARRAY = ARRAY;
        notifyDataSetChanged();
    }

    public Note getNoteAt(int position) {
        return ARRAY.get(position);
    }

    class NoteHolder extends RecyclerView.ViewHolder {
        private TextView textViewTitle;

        public NoteHolder(View itemView) {
            super(itemView);
            //Link UI With a var 1/2
            textViewTitle = itemView.findViewById(R.id.text_view_title);

            itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    int position = getAdapterPosition();
                    if (listener != null && position != RecyclerView.NO_POSITION) {
                        listener.onItemClick(ARRAY.get(position));
                    }
                }
            });
        }
    }

    public interface OnItemClickListener {
        //TODO Change
        void onItemClick(Note note);
    }

    public void setOnItemClickListener(OnItemClickListener listener) {
        this.listener = listener;
    }
}
```

### ADD data

GIST:
```java
//init RecyclerView
//TODO change recyclerView
RecyclerView recyclerView = findViewById(R.id.recyclerView);
recyclerView.setLayoutManager(new LinearLayoutManager(this));
recyclerView.setHasFixedSize(true);

//TODO change NoteAdapter
final NoteAdapter adapter = new NoteAdapter();
recyclerView.setAdapter(adapter);

//Listen onChanges
noteViewModel = ViewModelProviders.of(this).get(NoteViewModel.class);
noteViewModel.getAllProducts().observe(this, new Observer<List<Note>>() {
    @Override
    public void onChanged(@Nullable List<Note> notes) {
        //TODO change
        adapter.setNotes(notes);
        }
    });

btn.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        //TODO Change <arg1> <arg2> appropriate Ctor
        Note note = new Note(<arg1>, <arg2>);
        /TODO Change noteViewModel
        noteViewModel.insert(note);
        }
});
```

### Update Data

```java
//TODO Changes
Note note = new Note(<arg1>, <arg2>);
note.setId(id);
noteViewModel.update(note);
```

### Delete Data

```java
//TODO Change noteViewModel getNoteAt
//Should be used in Holder or swpie methods
noteViewModel.delete(adapter.getNoteAt(viewHolder.getAdapterPosition()));
```